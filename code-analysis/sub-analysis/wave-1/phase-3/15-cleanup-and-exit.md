# Wave 1 / Phase 3 / Task 7 (15): Shutdown Sequence and Cleanup

**Task**: Analyze shutdown sequence including cleanup, resource disposal, final saves, and exit codes.
**Analyst**: analyst-4
**Scope**: How the Claude Code CLI exits — the global shutdown pipeline (`src/utils/gracefulShutdown.ts` + `src/utils/cleanupRegistry.ts`), per-subsystem cleanup handlers, ordering constraints, exit codes, and failure modes.

---

## 1. Executive Summary

Claude Code has a **single, centralized exit pipeline** built around two small files:

- `src/utils/gracefulShutdown.ts` (529 LoC) — the ordered async pipeline that every code path converges on.
- `src/utils/cleanupRegistry.ts` (25 LoC) — a flat, unordered `Set<() => Promise<void>>` of cleanup callbacks contributed by every subsystem.

**Public entry points** (called from ~150 sites across the codebase):

| Entry point | Signature | Call style | Notes |
|---|---|---|---|
| `gracefulShutdown(code, reason?, options?)` | `async` | awaitable | main pipeline |
| `gracefulShutdownSync(code, reason?, options?)` | sync | fire-and-forget | schedules async path, sets `process.exitCode` immediately so callers can bail |
| `setupGracefulShutdown()` | memoized | install signal handlers | registers SIGINT/SIGTERM/SIGHUP + orphan-detect + uncaughtException/unhandledRejection |
| `forceExit(code)` | sync | `never` | `process.exit()` with `SIGKILL` fallback on EIO |
| `registerCleanup(fn)` | sync | returns unregister | subsystem opt-in |
| `runCleanupFunctions()` | `async` | internal | `Promise.all` on the whole set |
| `isShuttingDown()` | sync | predicate | used by callers to short-circuit |

**Key design choices**:

1. **Synchronous terminal cleanup happens first** (`cleanupTerminalModes()` at `gracefulShutdown.ts:436`), *before* any `await`. Alt-screen exit and the resume hint land on the main buffer even if the process is killed mid-cleanup.
2. **Cleanup registry is parallel, not serial** — every `registerCleanup` callback runs via `Promise.all` (`cleanupRegistry.ts:24`). Ordering across subsystems is NOT preserved. Ordering is instead encoded by *splitting cleanup into the graceful pipeline's discrete phases* (session-flush, registry, session-end hooks, analytics flush).
3. **Every async phase is time-bounded**. The registry has a 2 s cap, SessionEnd hooks have a user-configurable cap (default 1.5 s via `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`), analytics flush has 500 ms. A global failsafe at `max(5000, hookBudget + 3500)` ms guarantees `forceExit()` regardless.
4. **`forceExit()` has a `SIGKILL` fallback**: if `process.exit()` throws EIO (dead TTY after SIGHUP / SSH disconnect), it calls `process.kill(pid, 'SIGKILL')` to avoid hanging on a flush-to-dead-fd.
5. **Double-shutdown guard**: a module-scoped `shutdownInProgress` flag at `gracefulShutdown.ts:401` makes `gracefulShutdown()` idempotent.
6. **Bun signal-exit v4 pin** at `gracefulShutdown.ts:249-254` — a no-op `onExit(() => {})` keeps signal-exit v4's subscriber count above zero; without it, a Bun bug would have `removeListener` reset kernel sigactions and the SIGTERM handler silently stops firing.

**There are critical ordering constraints** — see §6.

---

## 2. The Shutdown Pipeline: Full Narrative

File: `src/utils/gracefulShutdown.ts:391-523`.

The async `gracefulShutdown(exitCode, reason, options)` runs in strict sequential order:

```
1. idempotency guard         (401-404)   shutdownInProgress ? return : set=true
2. resolve hook budget       (409-412)   dynamic import hooks.js to avoid cycle
3. arm failsafe timer        (417-426)   max(5000ms, hookBudget+3500ms), .unref()
4. set process.exitCode      (429)       for natural-exit observers
5. cleanupTerminalModes()    (436)       SYNCHRONOUS — runs before first await
6. printResumeHint()         (437)       SYNCHRONOUS — on main buffer
7. runCleanupFunctions()     (445-467)   capped at 2000ms via Promise.race
8. executeSessionEndHooks()  (472-480)   capped at sessionEndTimeoutMs
9. profileReport()           (483-487)   startup perf log (before analytics dies)
10. tengu_cache_eviction_hint (491-499)  before analytics flush
11. analytics flush          (504-511)   Promise.race([1P + Datadog, sleep(500)])
12. optional finalMessage    (513-520)   writeSync(2, ...)
13. forceExit(exitCode)      (522)       process.exit() or SIGKILL fallback
```

Each `try/catch` is written to *silently swallow errors and move on*. Shutdown that partially fails still exits cleanly — the failsafe timer is a last-resort hammer.

### 2.1 Why sync-first, then async

`cleanupTerminalModes()` and `printResumeHint()` use `writeSync` and run before the first `await`. The comment at line 431-435 is explicit: "ensures the hint is visible even if the process is killed during cleanup (e.g., SIGKILL during macOS reboot)". Async phases that come later (registry, hooks, analytics) would be cut short on host-initiated termination; the visible-to-user state (terminal mode reset + resume hint) is therefore committed *first*.

### 2.2 `cleanupTerminalModes()` — internal ordering (lines 59-136)

```
1. DISABLE_MOUSE_TRACKING                 (70)  FIRST — gives terminal round-trip
                                                to stop sending events while we
                                                unmount below.
2. Ink unmount on alt-screen instance     (86-95)
     - Runs final render on alt buffer
     - Ink's signal-exit hook writes 1049l exactly once
     - Fallback: manual writeSync(EXIT_ALT_SCREEN) if reconciler throws
3. inst.drainStdin()                      (98)  catches events that arrived
                                                during unmount tree-walk
4. inst.detachForShutdown()               (106) mark instance unmounted so
                                                signal-exit's deferred unmount
                                                early-returns, preventing
                                                redundant 1049l sequences
5. DISABLE_MODIFY_OTHER_KEYS + DISABLE_KITTY_KEYBOARD  (109-110)
6. DFE  (disable focus events)            (112)
7. DBP  (disable bracketed paste)         (114)
8. SHOW_CURSOR                            (116)
9. CLEAR_ITERM2_PROGRESS                  (119)
10. CLEAR_TAB_STATUS (if supported)       (121)
11. CLEAR_TERMINAL_TITLE                  (125-130)  unless CLAUDE_CODE_DISABLE_TERMINAL_TITLE
```

The comment at lines 72-85 is the *key* ordering note: you must exit alt-screen *before* writing the resume hint; otherwise the hint lands on the alt buffer and is lost when alt-screen exits. You must unmount Ink *through* its own mechanism (not by writing 1049l directly) because Ink has a signal-exit hook that will otherwise write 1049l a *second* time — the second `1049l` triggers DECRC and restores a cursor position that lands the shell prompt on the wrong line.

### 2.3 Resume hint (lines 144-184)

`printResumeHint()` runs *once* (a module-level flag `resumeHintPrinted` at line 138 guards re-entry from the failsafe timer). It only fires if:
- `process.stdout.isTTY`
- `getIsInteractive()` (bootstrap state)
- `!isSessionPersistenceDisabled()`
- `sessionIdExists(sessionId)` — so subcommands like `claude update` don't print the hint

The hint format is:
```
Resume this session with:
claude --resume <sessionId-or-"customTitle">
```

### 2.4 `forceExit()` (lines 193-232)

Last thing that runs. Key behaviors:
- Clears the failsafe timer.
- One more `instances.get(process.stdout)?.drainStdin()` (stdin bytes arriving between `DISABLE_MOUSE_TRACKING` and now).
- Calls `process.exit(exitCode)`. If that **throws** (Bun flushing stdout to a dead fd raises EIO on SIGHUP/SSH-disconnect), falls back to `process.kill(pid, 'SIGKILL')`.
- In tests (`NODE_ENV === 'test'`) `process.exit` is mocked to throw; the function re-throws so the test sees it.

### 2.5 Sync wrapper (lines 336-359)

`gracefulShutdownSync(code, reason, options)`:
- Sets `process.exitCode = code` immediately — this is the signal other code uses (`process.exitCode !== undefined`) to short-circuit further initialization. See `main.tsx:2312` ("If gracefulShutdown was initiated (e.g., user rejected trust dialog), skip all subsequent operations").
- Fires `gracefulShutdown()` and stores the promise in `pendingShutdown`.
- On rejection, falls back to `cleanupTerminalModes() → printResumeHint() → forceExit()`.
- `.catch(() => {})` at the tail prevents unhandled-rejection from the test-mode re-throw escaping.

---

## 3. Signal Handlers

All installed by the memoized `setupGracefulShutdown()` at `gracefulShutdown.ts:237-334`.

### 3.1 SIGINT (`:256-267`)
- **Print mode (`-p`/`--print`) is skipped**: `print.ts:1027-1034` installs its own SIGINT handler that aborts the in-flight AbortController, then calls `gracefulShutdown(0)`. If this global handler *also* ran, the two would race.
- Non-print: `gracefulShutdown(0)`.

### 3.2 SIGTERM (`:268-271`)
- `gracefulShutdown(143)` — exit code 128+15 by convention.

### 3.3 SIGHUP (`:273-276`, non-Windows only)
- `gracefulShutdown(129)` — exit code 128+1.

### 3.4 Orphan detection (`:281-296`, macOS TTY revocation)
- `setInterval(..., 30_000).unref()`. Checks `process.stdout.writable` + `process.stdin.readable`. When either becomes false (macOS revokes TTY fds instead of signaling SIGHUP), calls `gracefulShutdown(129)`.
- Skipped during scroll-drain (`getIsScrollDraining()`) to preserve event-loop ticks for scroll frames.

### 3.5 `uncaughtException` (`:301-310`)
- Logs name + first 2000 chars of message via `logForDiagnosticsNoPII` and `logEvent('tengu_uncaught_exception')`.
- **Does not** call `gracefulShutdown`. Node's default behavior (exit with code 1) still applies if no other handler exists, but since this is a registered handler, Node will *not* crash — the uncaught exception is effectively swallowed after logging.

### 3.6 `unhandledRejection` (`:313-333`)
- Logs name + first 2000 chars of message + first 4000 chars of stack.
- Also does not call `gracefulShutdown`.

### 3.7 Bun signal-exit v4 pin (`:254`)
- `onExit(() => {})` with no unsubscribe. Keeps v4's emitter subscriber count > 0 so `v4.unload()` never fires. Without this pin, any short-lived signal-exit v4 subscriber (execa child, Ink instance unmount) that happens to be the last subscriber causes `removeListener` to run on every signal — and under Bun, that resets the kernel sigaction, so SIGTERM silently falls back to terminate-by-default.

---

## 4. Cleanup Registry Participants

`src/utils/cleanupRegistry.ts` exposes a flat `Set`:

```ts
const cleanupFunctions = new Set<() => Promise<void>>()
export function registerCleanup(fn): () => void
export async function runCleanupFunctions() {
  await Promise.all(Array.from(cleanupFunctions).map(fn => fn()))
}
```

All participants run **in parallel** via `Promise.all` under the 2 s cap (`gracefulShutdown.ts:444-467`). There are **43 registering files** — below is a categorized directory.

### 4.1 Transport / external processes
| File:line | Description |
|---|---|
| `src/services/mcp/client.ts:1574` | per-MCP-server close; stdio transports escalate SIGINT→SIGTERM→SIGKILL with a 600 ms inner failsafe |
| `src/tasks/LocalShellTask/LocalShellTask.tsx:200,269` | kill shell child process; clear cleanup timeout |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx:507,549` | agent task cleanup |
| `src/tasks/LocalMainSessionTask.ts:116` | main session task cleanup |
| `src/utils/tmuxSocket.ts:314` | `killTmuxServer()` — tears down tmux sessions Claude started |
| `src/utils/bash/ShellSnapshot.ts:534` | `cleanupShellSnapshot()` |
| `src/utils/swarm/spawnInProcess.ts:183` (×2) | kill in-process swarm agents |
| `src/utils/swarm/backends/PaneBackendExecutor.ts:166` | kill pane-backed teammates |
| `src/entrypoints/init.ts:189,195` | shutdown LSP servers |
| `src/cli/remoteIO.ts:138,199` | close CCR client + transport |
| `src/bridge/replBridge.ts:1671` | `doTeardownImpl()` |
| `src/bridge/remoteBridgeCore.ts:746` | remote bridge teardown |
| `src/main.tsx:2493` | log `exited` diag event (sentinel: confirms cleanup ran) |

### 4.2 Persistence / state flushes
| File:line | Description |
|---|---|
| `src/utils/sessionStorage.ts:449` | flush + **re-append session metadata** to keep `customTitle`/`tag` in the 64 KB tail window for `--resume` |
| `src/utils/concurrentSessions.ts:66` | deregister PID file from `~/.claude/sessions/` |
| `src/history.ts:421` | close history DB |
| `src/utils/config.ts:904` | `reportConfigCacheStats()` — analytics, before flush |
| `src/utils/config.ts:1030` | stop fresh-config file watcher (`unwatchFile`) |
| `src/utils/sessionActivity.ts:103` | flush activity log |
| `src/utils/errorLogSink.ts:106` | flush error writer |
| `src/utils/debug.ts:190` | flush debug logs |

### 4.3 Watchers / detectors / locks
| File:line | Description |
|---|---|
| `src/utils/hooks/fileChangedWatcher.ts:39` | dispose file watcher |
| `src/utils/settings/changeDetector.ts:93` | dispose settings detector |
| `src/utils/skills/skillChangeDetector.ts:138` | stop skill watcher |
| `src/keybindings/loadUserBindings.ts:403` | `disposeKeybindingWatcher()` |
| `src/utils/cronTasksLock.ts:95` | release cron lock |
| `src/utils/computerUse/computerUseLock.ts:96` | release Computer Use lock |
| `src/services/teamMemorySync/watcher.ts:228` | `stopTeamMemoryWatcher()` |
| `src/utils/git/gitFilesystem.ts:377` | dispose git filesystem |
| `src/services/preventSleep.ts:115` | kill `caffeinate` subprocess |
| `src/services/remoteManagedSettings/index.ts:627` | stop managed-settings polling |
| `src/services/policyLimits/index.ts:651` | stop policy-limits polling |

### 4.4 Output / telemetry
| File:line | Description |
|---|---|
| `src/utils/telemetry/instrumentation.ts:561,698` | OTEL `shutdownTelemetry` (meter provider + flush) |
| `src/utils/telemetry/perfettoTracing.ts:308` | flush Perfetto traces |
| `src/utils/streamJsonStdoutGuard.ts:93` | flush any buffered NDJSON |
| `src/utils/asciicast.ts:233` | close cast recording |
| `src/utils/terminalPanel.ts:127` | dispose terminal panel |
| `src/cli/print.ts:1038` | diagnostic dump: `run_active`, `run_phase`, `worker_status`, background-task counts — intentionally fires on SIGTERM so stuck-session healthsweep can name the `do/while(waitingForAgents)` poll |

### 4.5 Install / bootstrap
| File:line | Description |
|---|---|
| `src/utils/nativeInstaller/installer.ts:1118` | cleanup install state |
| `src/utils/plugins/headlessPluginInstall.ts:164` | cleanup plugin install cache |
| `src/upstreamproxy/upstreamproxy.ts:135` | stop upstream proxy |

---

## 5. Exit Points and Codes

### 5.1 Exit-code conventions

| Code | Meaning | Source |
|---|---|---|
| `0` | Normal / success | default |
| `1` | Error, validation failure, permission issue, trust rejection | ~50 sites in `print.ts`, ~30 in `main.tsx` |
| `129` | SIGHUP or TTY-orphan detection | `gracefulShutdown.ts:275, 292` — 128+1 |
| `143` | SIGTERM | `gracefulShutdown.ts:270` — 128+15 |
| `undefined` | Still running (or `gracefulShutdownSync` used as a bail signal) | set by `process.exitCode = code` |

### 5.2 Bare `process.exit()` (bypasses the pipeline)

`main.tsx` contains **many** direct `process.exit(1)` calls in the very-early bootstrap path (lines 270, 442, 468, 481, 494, 1172-1858, etc.). The comment at `main.tsx:267-269` explains why:

> *"Use process.exit directly here since we're in the top-level code before imports and gracefulShutdown is not yet available."*

These early exits happen before ink/analytics/MCP have started, so there is nothing to clean up. Safe.

Other places use `process.exit()` *after* awaiting `gracefulShutdown()`:
- `main.tsx:3285-3286, 3302-3303, 3441-3442, 3512-3513, 3548` — `await gracefulShutdown(0); process.exit(0)`. The trailing `exit` is a belt-and-suspenders: `gracefulShutdown` already calls `forceExit()`, but if a test-mode mock returned, `exit(0)` completes the exit.

### 5.3 ExitReason enum (`src/entrypoints/sdk/coreSchemas.ts:747-754`)

```ts
export const EXIT_REASONS = [
  'clear',                       // /clear slash command
  'resume',                      // resume session
  'logout',                      // explicit logout
  'prompt_input_exit',           // Ctrl-D or prompt-level exit
  'other',                       // default
  'bypass_permissions_disabled', // permission mode change invalidates session
] as const
```

Propagated via `gracefulShutdown(..., reason)` → `executeSessionEndHooks(reason, ...)` → `SessionEnd` hook input (`utils/hooks.ts:4097-4141`). Lets user SessionEnd hooks discriminate why the session ended.

---

## 6. Critical Ordering Constraints

**Flag**: The cleanup pipeline **does** have critical ordering requirements. They are enforced structurally (by pipeline phases), not by hand-rolled async chains.

| Ordering | Why | Where it's enforced |
|---|---|---|
| Terminal mode reset **before** any await | Ensures terminal is sane even if async cleanup is SIGKILL'd mid-flight | `gracefulShutdown.ts:436-437` (pre-await block) |
| `DISABLE_MOUSE_TRACKING` **before** Ink unmount | Terminal needs round-trip to stop sending events — doing it during unmount tree-walk wastes that time | `gracefulShutdown.ts:70` + comment 64-69 |
| Exit alt-screen **before** `printResumeHint` | Hint must land on main buffer, not alt | `gracefulShutdown.ts:71-85` comment |
| Ink.unmount() **instead of** manual `writeSync(EXIT_ALT_SCREEN)` | Ink has its own signal-exit hook; writing 1049l twice triggers DECRC twice, moving the cursor back over the resume hint | `gracefulShutdown.ts:72-94` |
| `detachForShutdown()` **after** unmount | Prevents signal-exit's deferred unmount from writing redundant 1049l sequences that clobber the resume hint on tmux | `gracefulShutdown.ts:99-106` |
| Session flush + registry **before** SessionEnd hooks | User hooks can be slow or hang; session data (transcript, metadata) must be persisted first so `--resume` works even if the failsafe fires during the hook phase | `gracefulShutdown.ts:442-480` |
| `reAppendSessionMetadata()` **after** flush | `readLiteMetadata` reads only the last 64 KB of the transcript; without re-append, a post-`/rename` session with lots of appended messages would show the auto-generated firstPrompt instead of the custom title | `sessionStorage.ts:449-462` |
| `profileReport()` **before** analytics shutdown | Analytics shutdown cancels timers that `profileReport` depends on | `gracefulShutdown.ts:482-487` |
| `tengu_cache_eviction_hint` event **before** analytics flush | Event must reach the pipeline before exporters are told to flush | `gracefulShutdown.ts:489-499` |
| Analytics flush **capped at 500 ms** | Prior to the cap, the 1P exporter awaited all pending `axios.POST`s (10s each), eating the full failsafe budget on slow networks | `gracefulShutdown.ts:501-511` + comment |
| Failsafe budget **after** hook budget is resolved | Without this ordering, a user-configured 10 s hook budget would be silently truncated by the 5 s failsafe (per `gh-32712 follow-up` comment) | `gracefulShutdown.ts:406-423` |
| MCP stdio `SIGINT → SIGTERM → SIGKILL` escalation | Docker and other MCP servers rely on specific signals for graceful shutdown; `StdioClientTransport.close()` only sends abort | `src/services/mcp/client.ts:1429-1562` |
| `forceExit()` **always** last | Catches the case where a cleanup handler hangs past the failsafe | `gracefulShutdown.ts:522` + failsafe `:417-426` |

**Non-ordered (by design)**: Everything inside `runCleanupFunctions()` runs via `Promise.all`. Subsystems that need internal ordering must encode it inside their own callback (see MCP stdio escalation above) — *not* rely on registration order. The cleanup set is a `Set<Promise>`, so registration order isn't even preserved.

---

## 7. Graceful vs Forced Exit

### 7.1 Normal graceful exit
User types `/exit`, presses Ctrl-D, or completes a `-p` query:
1. Some caller (REPL handler, `print.ts:runHeadless`, `main.tsx:initOnly`) invokes `gracefulShutdown(0, 'prompt_input_exit')`.
2. Full 13-step pipeline runs. Typical wall-clock: ~100-500 ms depending on MCP count + hook budget.
3. `forceExit(0)` → `process.exit(0)`.

### 7.2 Signal-triggered exit
SIGINT / SIGTERM / SIGHUP delivered. Handler converts to `gracefulShutdown(code)`. Same pipeline, same bounded timeouts.

### 7.3 Failsafe-triggered exit (hung cleanup)
Failsafe timer at `gracefulShutdown.ts:417-426` fires after `max(5000, hookBudget + 3500)` ms:
1. `cleanupTerminalModes()` — restore terminal even though pipeline hung
2. `printResumeHint()` — only if not already printed
3. `forceExit(code)` — which also clears the failsafe timer to be safe

### 7.4 TTY-dead exit (SSH disconnect, macOS terminal close)
- SIGHUP on Linux, TTY fd revocation on macOS.
- Pipeline runs, but `writeSync` calls in `cleanupTerminalModes` may throw — caught by the outer `try` (line 132-135).
- Session flush and registry run; `writeSync(2, finalMessage)` catches EPIPE.
- `forceExit()`'s `process.exit()` throws EIO → **falls back to `process.kill(pid, 'SIGKILL')`** (line 222). This is the critical escape hatch.

### 7.5 MCP-hung-cleanup escalation (`src/services/mcp/client.ts:1429-1562`)
Inside a single MCP's cleanup callback:
1. `SIGINT` at t=0
2. Poll every 50 ms for process exit
3. If still alive at t=100 ms → `SIGTERM`
4. If still alive at t=500 ms → `SIGKILL`
5. Absolute failsafe at t=600 ms stops monitoring (child is then orphaned; OS will reap)

This is the *only* subsystem in the codebase that does signal escalation in cleanup. All other registered cleanups rely on the outer 2 s Promise.race cap.

---

## 8. How Per-Mode Exit Paths Work

### 8.1 Interactive REPL (`main.tsx`)
- Ink tree mounted.
- `setupGracefulShutdown()` registered via first render path.
- User Ctrl-C → SIGINT handler (`:256`) → `gracefulShutdown(0)`.
- User `/exit` or Ctrl-D → slash command / ExitAppHandler → `gracefulShutdownSync(0, 'prompt_input_exit')`.
- Trust rejected → `gracefulShutdownSync(1)` (e.g., `launchInvalidSettingsDialog` `onExit` at `main.tsx:2333`).

### 8.2 Headless `-p` / `--print` (`cli/print.ts`)
- Global SIGINT handler is **skipped** at `gracefulShutdown.ts:262-264`.
- `print.ts:1027-1034` registers its own SIGINT handler that aborts the in-flight query's AbortController first, *then* calls `gracefulShutdown(0)`. This preserves the query's `catch(AbortError)` path (which emits the final error result to stdout).
- `runHeadless()` ends with `gracefulShutdownSync(lastMessage?.is_error ? 1 : 0)` at `print.ts:971-973`.
- ~50 other `gracefulShutdownSync(1)` callsites for validation / configuration errors.
- Diagnostic dump registered at `print.ts:1038` captures run phase, worker status, and background-task counts on shutdown — specifically so a stuck `do/while(waitingForAgents)` poll can be diagnosed without parsing the transcript.

### 8.3 `--init-only` subcommand
- `main.tsx:2580`: `gracefulShutdownSync(0)` immediately after setup + session-start hooks.

### 8.4 Remote SSH / bridge modes
- `main.tsx:3173, 3239, 3275, 3282, 3320, 3406, 3416, 3429, 3459, 3577, 3619, 3661` all use `await exitWithError(root, msg, () => gracefulShutdown(code))`. The `exitWithError` helper mounts a final Ink error screen, waits briefly for render, then awaits the shutdown.
- Bridge-specific cleanup: `replBridge.ts:1671`, `remoteBridgeCore.ts:746`, `remoteIO.ts:138,199`.

### 8.5 Subcommand paths (e.g., `claude update`)
- Use bare `process.exit()` at the top level before `gracefulShutdown` is available.
- Do **not** write the resume hint (`sessionIdExists` returns false — there is no session file).

---

## 9. Exit Code Table — Full Inventory

Gathered from `rg gracefulShutdownSync\|process\.exit\b` across codebase:

| Exit code | Used by (representative) |
|---|---|
| `0` | `initOnly`, normal REPL exit, successful `-p` run, `process.exit(0)` after `await gracefulShutdown(0)` |
| `1` | Validation errors, trust rejection, SDK-option conflicts, invalid `--resume`, failed auth, failed migration, CI setup failure — ~80 sites |
| `129` | SIGHUP, orphan detection |
| `143` | SIGTERM |
| `url-scheme-provided` | `main.tsx:675`: `process.exit(urlSchemeResult ?? 1)` — delegates to URL scheme handler |

No other codes are used. `128+SIGN` is observed only for 129 / 143.

---

## 10. Integration Points (Inputs and Outputs)

### 10.1 Who calls shutdown?
- Signal handlers (SIGINT/SIGTERM/SIGHUP + orphan) — see §3
- REPL slash commands (`/exit`, `/logout`, `/clear`) — route through REPL handlers that call `gracefulShutdownSync`
- `print.ts` — ~50 sites
- `main.tsx` — ~30 sites
- Trust dialog, invalid settings dialog, other `onExit` callbacks
- Bridge teardown and remote-session lifecycle

### 10.2 What shutdown reads
- `process.exitCode` — to detect re-entry
- `process.argv` — detect `-p`/`--print` for SIGINT skip
- `process.stdout.isTTY`, `process.stdin.readable` — for orphan detection and terminal cleanup gating
- `bootstrap/state.ts`: `getIsInteractive()`, `getIsScrollDraining()`, `getLastMainRequestId()`, `getSessionId()`, `isSessionPersistenceDisabled()`
- `ink/instances.ts` — the Ink instance map, for unmount + drainStdin
- `sessionStorage` — `getCurrentSessionTitle()`, `sessionIdExists()` for resume hint
- Env vars: `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`, `CLAUDE_CODE_DISABLE_TERMINAL_TITLE`, `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER`, `NODE_ENV`

### 10.3 What shutdown writes / emits
- Terminal: alt-screen exit, cursor, title, focus/paste/kitty-keyboard resets
- stdout: resume hint (via `writeSync`)
- stderr: optional `finalMessage`
- Diagnostics: `shutdown_signal` events (SIGINT / SIGTERM / SIGHUP / orphan_detected / uncaught_exception / unhandled_rejection)
- Analytics: `tengu_uncaught_exception`, `tengu_unhandled_rejection`, `tengu_cache_eviction_hint`
- Session transcript: final flush + `reAppendSessionMetadata`
- OTEL: `shutdownTelemetry` flushes pending spans/metrics

### 10.4 Lifecycle relative to main()
- Signal handlers registered lazily (`setupGracefulShutdown` is memoized, called from first REPL render).
- Registry accrues callbacks throughout startup — MCP clients, task instances, bridges, watchers, locks all self-register on creation.
- Shutdown pipeline is idempotent (`shutdownInProgress` guard at `:401-404`).
- `resetShutdownState()` exists only for tests (`:372-380`).

---

## 11. Complexity Hotspots

| Area | Complexity | Why |
|---|---|---|
| `cleanupTerminalModes()` internal ordering | **High** | Interaction between Ink's signal-exit hook, DECSET/DECRST double-write bugs, and multiplexer (tmux/screen) cursor-save semantics. Well-documented via inline comments 72-106; one wrong move and the resume hint lands on the wrong buffer or the shell prompt ends up on a bad line. |
| MCP stdio SIGINT/SIGTERM/SIGKILL escalation | **High** | Nested promise + setInterval + setTimeout with multiple resolve paths. 100+ lines of signal-escalation state machine inside one anonymous function. |
| signal-exit v4 Bun bug workaround | **High cognitive load, low code volume** | The one-line `onExit(() => {})` at line 254 + 18-line comment explaining what breaks without it. Genuinely subtle. |
| Failsafe budget calculation | **Medium** | `max(5000, hookBudget + 3500)` was a fix for the hook-budget-truncation bug (gh-32712 follow-up). Dynamic import of `hooks.js` at `:409-412` to resolve the hook budget without creating a module cycle with `gracefulShutdown.ts`. |
| `forceExit()` `SIGKILL` fallback | **Medium** | The EIO path is rarely exercised in local testing but is the only thing preventing SSH/TTY-dead sessions from hanging. Catches `process.exit()` itself throwing. |
| SessionEnd hook budget truncation | **Medium** | User hooks can set `hook.timeout`; capped by `getSessionEndHookTimeoutMs()` at `utils/hooks.ts:176-182`, budget propagates to both `AbortSignal.timeout` and `timeoutMs` option. |
| `reAppendSessionMetadata` after flush | **Medium** | The 64 KB tail window semantics of `readLiteMetadata` are subtle; the comment at `sessionStorage.ts:450-455` is load-bearing. |

---

## 12. Integration with Other Subsystems

### 12.1 Session persistence (task #6)
- Shutdown drives the *final* flush at `sessionStorage.ts:449` + metadata re-append.
- Relies on session persistence being idempotent — `project?.flush()` and `project?.reAppendSessionMetadata()` are safe to call multiple times.
- Session persistence sets `cleanupRegistered` flag (`sessionStorage.ts:441`) to register exactly once.

### 12.2 MCP (task #2)
- Per-server cleanup registered in `client.ts:1574`.
- Stdio escalation is the most sophisticated signal logic in any cleanup handler.
- In-process servers (Chrome MCP, Computer Use) have simpler `inProcessServer.close() + client.close()` cleanup (lines 1406-1418).

### 12.3 Hooks (task #1 follow-up)
- `executeSessionEndHooks()` at `hooks.ts:4097` is the SessionEnd entry point.
- Budget via `getSessionEndHookTimeoutMs()` (default 1500 ms, env-overridable).
- Hook failures write to stderr (line 4128-4133) because Ink is already unmounted by shutdown time.

### 12.4 Analytics / telemetry
- `shutdown1PEventLogging()` (`firstPartyEventLogger.ts:116`)
- `shutdownDatadog()` (`datadog.ts:151`)
- OTEL registered separately via `registerCleanup` at `telemetry/instrumentation.ts:561, 698`.
- 500 ms cap on 1P+Datadog protects against network stalls.

### 12.5 Interruption handling (task #5, adjacent report)
- SIGINT → `gracefulShutdown(0)` in non-print mode.
- In print mode, `print.ts:1027` handles SIGINT by aborting in-flight query + calling `gracefulShutdown(0)`. The query's `AbortError` path is what emits the final error result to stdout.

### 12.6 Ink/terminal (bootstrap)
- `ink/instances.ts` tracks the mounted Ink instance per stdout.
- `Ink#unmount()` is hooked via `onExit()` from signal-exit; `detachForShutdown()` (`ink.tsx:932`) is the handshake that tells Ink "don't run your unmount again, we already did it manually."

### 12.7 Daemon / workers (task #8)
- Not checked in detail for this report — daemon lifecycle has its own cleanup machinery separate from `cleanupRegistry`. See report #16 (daemon mode) for follow-up.

---

## 13. Recommendations / Follow-ups

1. **Verify daemon-mode shutdown path** (task #16): does it use the same `gracefulShutdown` pipeline, or is there a separate daemon shutdown routine? Specifically check `daemon/main.js` and `daemon/workerRegistry.js`.
2. **Background task termination on agent exit**: `killShellTasksForAgent` at `LocalShellTask/killShellTasks.ts:53` is called from `runAgent.ts` finally block. This is orthogonal to the global cleanup registry — verify that a global shutdown mid-agent also kills these shells (currently they're individually registered via the task's `unregisterCleanup`).
3. **Bridge teardown order**: `replBridge.ts` and `remoteBridgeCore.ts` both register cleanup. Verify whether these need ordering between themselves and with MCP (both use transports — could be a race when shutting down simultaneously).
4. **Resume hint correctness**: the hint is printed *before* session flush. If flush *fails*, the user is told to `--resume <id>` but the session file may be inconsistent. Trade-off is intentional (resume hint visibility beats correctness), but worth documenting for the main codebase report.
5. **Analytics budget of 500 ms vs actual network conditions**: events queued during the last ~500 ms of the session may be dropped on slow networks. This is explicitly acknowledged in the comments (`:501-503`) but may surface as missing telemetry on stuck-session reports.
6. **Unchecked subsystems** (found in registry scan but not investigated in detail):
   - `src/utils/asciicast.ts:233` — session recording closure
   - `src/utils/swarm/*` — swarm teammate cleanup
   - `src/upstreamproxy/upstreamproxy.ts:135` — proxy shutdown
   - `src/entrypoints/init.ts:189,195` — LSP server shutdown
   Recommend a follow-up that enumerates each subsystem's cleanup callback and verifies it completes in well under 2 s.
7. **Test coverage**: `resetShutdownState()` exists at `:372-380` for tests. A focused test matrix would verify (a) double-call idempotency, (b) failsafe-fired path, (c) forceExit EIO → SIGKILL path, (d) print-mode SIGINT skip.

---

## Appendix A — Key Files and LoC

| File | LoC | Role |
|---|---|---|
| `src/utils/gracefulShutdown.ts` | 529 | central pipeline |
| `src/utils/cleanupRegistry.ts` | 25 | registry primitive |
| `src/utils/hooks.ts` (SessionEnd) | `:4097-4141` | SessionEnd hook invocation |
| `src/utils/hooks.ts` (budget) | `:176-182` | `getSessionEndHookTimeoutMs` |
| `src/services/mcp/client.ts` | `:1404-1574` | MCP per-server cleanup + signal escalation |
| `src/tasks/LocalShellTask/killShellTasks.ts` | 76 | agent-scoped shell-task killer |
| `src/utils/sessionStorage.ts` | `:440-467, :1583-1585` | session flush + metadata re-append registration |
| `src/cli/print.ts` | `:1024-1049` | print-mode SIGINT handler + diagnostic dump on shutdown |
| `src/main.tsx` | `:2308-2315, :2493` | exitCode short-circuit + `exited` diag cleanup |
| `src/entrypoints/sdk/coreSchemas.ts` | `:747-756` | `EXIT_REASONS` enum |

## Appendix B — Timeout Inventory (all of shutdown)

| Phase | Cap | Override |
|---|---|---|
| Registry (`runCleanupFunctions`) | 2000 ms | none |
| SessionEnd hooks (per-hook + total) | 1500 ms default | `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` env |
| MCP stdio SIGINT → SIGTERM | 100 ms | none |
| MCP stdio SIGTERM → SIGKILL | 400 ms | none |
| MCP stdio monitoring absolute | 600 ms | none |
| Analytics flush (1P + Datadog) | 500 ms | none |
| **Failsafe (global)** | `max(5000, hookBudget + 3500)` ms | indirect via hook budget |

All timeouts except the outer failsafe are enforced via `Promise.race([op, sleep(ms)])` or `setTimeout` — and in every case the outer failsafe at `max(5000, hookBudget + 3500)` is the last-resort guarantee.
