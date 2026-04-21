# Concurrent Processes Running Alongside the REPL

**Task**: #4 — Analyze concurrent processes running alongside REPL
**Scope**: What runs in parallel with the main REPL loop — pre-render prefetches, post-render prefetches, background housekeeping, watchers, MCP connections, bridge/remote, and cron.
**Primary files**: `src/main.tsx`, `src/screens/REPL.tsx`, `src/utils/backgroundHousekeeping.ts`, `src/interactiveHelpers.tsx`, `src/setup.ts`, `src/services/autoDream/autoDream.ts`, `src/services/preventSleep.ts`, `src/services/mcp/*`, `src/hooks/useReplBridge.tsx`, `src/hooks/useRemoteSession.ts`, `src/hooks/useScheduledTasks.ts`, `src/utils/hooks/fileChangedWatcher.ts`, `src/utils/settings/changeDetector.ts`.

---

## 1. Top-Level Picture

The REPL is the foreground React/Ink tree, but a ring of concurrent activity runs outside of it. I group them by **when they start** and **how long they live**:

1. **Pre-render prefetches** (`setup.ts`) — begin inside `setup()`, fire-and-forget (`void ...`), consumed later by REPL or first API call.
2. **First-render deferred prefetches** (`startDeferredPrefetches()` in `src/main.tsx:388-431`) — run immediately after `root.render(element)` in `src/interactiveHelpers.tsx:100`.
3. **After first user submit** (`startBackgroundHousekeeping()` in `src/utils/backgroundHousekeeping.ts`) — mounted inside the REPL via `useEffect` at `src/screens/REPL.tsx:3903-3907`, gated on `submitCount === 1`.
4. **Per-session long-lived watchers & timers** — started at various points and kept alive with `.unref()`ed timers or disposed via `registerCleanup()`.
5. **Per-turn transient tasks** — spawned by tools (BashTool / LocalShellTask / LocalAgentTask / AgentTool) and teardown on tool completion.
6. **External peers** — MCP servers, bridge/remote WebSocket peers, cron-fired prompts, UDS messaging inbox.

All of them run on the same event loop (Node/Bun single-threaded runtime) — "concurrent" here means overlapping async work, not multithreading. Heavy CPU work (analytics batching, event-loop stall detection, etc.) lives behind subprocesses or unref'd intervals to keep the REPL render path clear.

---

## 2. Pre-Render Work Kicked From `setup()`

These begin inside `src/setup.ts` and race with the REPL render.

| What | Where | Lifetime | Notes |
|---|---|---|---|
| UDS messaging server | `setup.ts:95-101` → `utils/udsMessaging.js` (dynamic import) | Process lifetime | Bound *before* any `SessionStart` hook snapshots env, so `CLAUDE_CODE_MESSAGING_SOCKET` is visible to hook subprocesses. Gated: `!isBareMode() || messagingSocketPath !== undefined`, plus `feature('UDS_INBOX')`. |
| FileChanged / CwdChanged chokidar watcher | `setup.ts:172` → `src/utils/hooks/fileChangedWatcher.ts:28-46` | Process lifetime | `awaitWriteFinish: {stabilityThreshold: 500, pollInterval: 200}`. Self-disposes via `registerCleanup`. Only starts if `CwdChanged`/`FileChanged` arrays are non-empty in the hook snapshot. |
| Version lock | `setup.ts:303` → `utils/nativeInstaller/installer.ts:1048` | Filesystem marker | Fire-and-forget; prevents another claude from deleting the running binary. |
| Session memory hook registration | `setup.ts:294` | Lazy | Sync registration only; gate check deferred. |
| ContextCollapse init | `setup.ts:296-301` (sync `require`, gated on `CONTEXT_COLLAPSE`) | Lazy | |
| Commands prefetch | `setup.ts:321-323` — `void getCommands(getProjectRoot())` | Memoized | Memoized cache is joined at `src/main.tsx:2029` via `Promise.all`. |
| Plugin hooks prefetch | `setup.ts:324-329` — `loadPluginHooks()` + `setupPluginHookHotReload()` | Hot-reload stays alive | Consumed by `processSessionStartHooks` before render. |
| Ant-only commit attribution + team memory + session file access hooks | `setup.ts:337-369` | Hook registrations | `registerAttributionHooks()` wrapped in `setImmediate` to delay git subprocess spawn until *after* first render. |
| Analytics/error sinks | `setup.ts:371` — `initSinks()` | Process lifetime | Drains any `logEvent`s queued before attach. |
| API key helper prefetch | `setup.ts:380` — `prefetchApiKeyFromApiKeyHelperIfSafe` | Once | Only runs when trust already confirmed. |

---

## 3. Deferred Prefetches (`startDeferredPrefetches`)

Defined at `src/main.tsx:388-431`, called from:

- `src/interactiveHelpers.tsx:100` — after `root.render(element)` in `renderAndRun` (the main interactive path).
- `src/main.tsx:2816-2817` — also in the headless/print path (runs "immediately" because there is no user-typing window to hide the work).

**Early returns**: `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` truthy **or** `isBareMode()` → skip the whole block. This keeps `--bare` clean and makes startup profiles reproducible.

Parallel fire-and-forget kicks:

```
void initUser()                           // Statsig / growthbook identity init
void getUserContext()                     // CLAUDE.md directory walk (memoized)
prefetchSystemContextIfSafe()             // getSystemContext (git status/branch/log), gated on trust
void getRelevantTips()                    // Tips engine pre-warm
void prefetchAwsCredentialsAndBedRockInfoIfSafe()   // if BEDROCK
void prefetchGcpCredentialsIfSafe()                 // if VERTEX
void countFilesRoundedRg(cwd, AbortSignal.timeout(3000), [])  // ripgrep file count for prompts
void initializeAnalyticsGates()           // updates gate cache from server
void prefetchOfficialMcpUrls()            // MCP registry
void refreshModelCapabilities()           // model metadata
void settingsChangeDetector.initialize()  // chokidar + MDM poll (see §6)
void skillChangeDetector.initialize()     // if !isBareMode()
void import('./utils/eventLoopStallDetector.js').then(m => m.startEventLoopStallDetector())  // ant-only
```

Several are memoized (`getUserContext`, `getSystemContext`, `refreshModelCapabilities`) and designed so that the later blocking read in `print.ts`/`query.ts` becomes a cache hit.

---

## 4. Background Housekeeping (`startBackgroundHousekeeping`)

Defined at `src/utils/backgroundHousekeeping.ts:31-94`. Called from:

- `src/screens/REPL.tsx:3903-3907` — `useEffect(() => { if (submitCount === 1) startBackgroundHousekeeping(); }, [submitCount])`. **Waits for the first user submit**, then fires exactly once.
- `src/main.tsx:2818` — headless path, started right after `startDeferredPrefetches` (no user typing window).

What runs:

- `void initMagicDocs()` — MagicDocs
- `void initSkillImprovement()` — `utils/hooks/skillImprovement.ts`
- `feature('EXTRACT_MEMORIES')` → `extractMemoriesModule.initExtractMemories()`
- `initAutoDream()` — mounts the runner; `runAutoDream` walks sessions, honors a time-gate (`cfg.minHours`) and a `SESSION_SCAN_INTERVAL_MS` throttle (see `src/services/autoDream/autoDream.ts:122-170`).
- `void autoUpdateMarketplacesAndPluginsInBackground()` — pulls plugins/marketplaces.
- `feature('LODESTONE')` + interactive → `ensureDeepLinkProtocolRegistered()`.
- `setTimeout(runVerySlowOps, DELAY_VERY_SLOW_OPERATIONS_THAT_HAPPEN_EVERY_SESSION /*10min*/).unref()` — schedules `cleanupOldMessageFilesInBackground()` and `cleanupOldVersions()`. **Reschedules itself** if `getLastInteractionTime() > Date.now() - 60s` (don't make a typing user wait for I/O).
- Ant-only: `setInterval(..., 24h).unref()` → `cleanupNpmCacheForAnthropicPackages()` + `cleanupOldVersionsThrottled()`.

The key pattern here: **`.unref()` every timer/interval** so these cannot keep the process alive. The "did the user touch the keyboard in the last 60s" re-arm is the only admission-control mechanism against slow I/O.

---

## 5. Per-Session Long-Lived Peers

### 5a. Analytics / error sinks
- `src/services/analytics/sink.ts:109` — `initializeAnalyticsSink()`. Attach point for the in-memory `logEventImpl` + `logEventAsyncImpl`. Events enqueued before attach are drained.
- `src/utils/errorLogSink.ts` — attaches to `logError()`.

### 5b. `preventSleep` (macOS only)
`src/services/preventSleep.ts:36-58`. Ref-counted `caffeinate` subprocess, timeout 300s, restart interval 240s. Self-healing on SIGKILL: the child auto-exits after the timeout. Reference increments/decrements are driven by the query lifecycle (`startPreventSleep()` on query begin, `stopPreventSleep()` on end).

### 5c. Settings change detector
`src/utils/settings/changeDetector.ts` — single chokidar watcher over every settings source (`SETTING_SOURCES`) plus MDM poll timer at `MDM_POLL_INTERVAL_MS = 30 min`.

Notable constants:
- `FILE_STABILITY_THRESHOLD_MS = 1000`
- `FILE_STABILITY_POLL_INTERVAL_MS = 500`
- `INTERNAL_WRITE_WINDOW_MS = 5000` — drops events that happen within 5s of `markInternalWrite()` so the claude's own config-edits don't retrigger the reload.
- `DELETION_GRACE_MS` — handles delete-then-recreate from auto-updates.

On change, `executeConfigChangeHooks()` fires; a blocking result can stop propagation. `settingsChanged` signal notifies subscribers.

### 5d. Skill change detector
`src/utils/skills/skillChangeDetector.ts` — parallel structure to settings detector.

### 5e. FileChanged hook watcher
See §2.

### 5f. Event loop stall detector (ant-only)
`src/utils/eventLoopStallDetector.js` — logs when the event loop is blocked >500 ms. Only starts under `"external" === 'ant'`.

### 5g. SDK heap dump monitor (ant-only)
`src/utils/sdkHeapDumpMonitor.js` — started at `src/main.tsx:2819-2821`. Pairs with heap-dump scheduler for memory-leak investigations.

### 5h. Concurrent sessions registry
`src/utils/concurrentSessions.ts` — writes a PID + metadata file under `~/.claude/sessions/` on startup. Types: `'interactive' | 'bg' | 'daemon' | 'daemon-worker'`. Reads/writes on `onSessionSwitch` so `--resume` keeps the PID file in sync. The file is the coordination surface for `claude --bg` and daemon supervisors; nothing "polls" it from within the REPL — it's a cleanup/claim marker.

---

## 6. MCP Connections

**Wiring**:
- `src/services/mcp/client.ts` (3348 LOC) — the connection pool, protocol handshake, tool/resource/command discovery.
- `src/services/mcp/MCPConnectionManager.tsx` — React context wrapper exposing `reconnectMcpServer` / `toggleMcpServer`.
- `src/services/mcp/useManageMCPConnections.ts` — hook driving the pool.
- Each MCP server = a persistent stdio/HTTP/SDK transport + a JSON-RPC client.

**Concurrency**:
- At startup, `prefetchOfficialMcpUrls()` runs in `startDeferredPrefetches`.
- For each configured MCP server, `useManageMCPConnections` launches a connection. Servers run as **subprocesses** (stdio) or remote connections (HTTP/SSE/WebSocket), truly parallel to the REPL.
- `claude.ai` connector has a bounded wait at `src/main.tsx:2800-2808` (`CLAUDE_AI_MCP_TIMEOUT_MS`) — if the connector isn't ready, we proceed; the connection continues in the background.
- Rare case: in-process SDK MCP transports (`src/services/mcp/InProcessTransport.ts`, `SdkControlTransport.ts`) run on the same loop but under distinct message channels.

See also task #2 (MCP initialization) for the protocol-level view; this task scopes concurrency only.

### Flag the follow-up
The MCP client is 3348 LOC and exposes tools, commands, resources, sampling, elicitation, and OAuth. A dedicated connection-pool deep-dive would complement task #2's init analysis.

---

## 7. Bridge / Remote / Direct-Connect / SSH (REPL-hosted peers)

Each of these runs inside the React tree but maintains a long-lived external connection:

| Hook | File | Transport | Lifecycle |
|---|---|---|---|
| `useReplBridge` | `src/hooks/useReplBridge.tsx` | claude.ai WebSocket via `replBridgeTransport` | Mounted once per REPL; `replBridgeEnabled` state flips it on/off. Circuit-breaker `consecutiveFailuresRef` + `MAX_CONSECUTIVE_INIT_FAILURES`. Inbound messages queued via `queuedCommands`. |
| `useRemoteSession` | `src/hooks/useRemoteSession.ts` | CCR WebSocket (`src/remote/SessionsWebSocket.ts`) | `--remote` mode; takes over setMessages/setIsLoading. |
| `useDirectConnect` | `src/hooks/useDirectConnect.ts` | WebSocket to a `claude` server (`src/server/directConnectManager.ts`) | `claude connect`. |
| `useSSHSession` | `src/hooks/useSSHSession.ts` | SSH child process stdin/stdout | `claude ssh`. |

A single state `activeRemote = sshRemote.isRemoteMode ? sshRemote : directConnect.isRemoteMode ? directConnect : remoteSession` selects which one receives messages (`src/screens/REPL.tsx:1421-1422`). The others stay dormant.

`initReplBridge`, `sessionRunner`, `replBridgeHandle` (all under `src/bridge/`) are the non-React guts. `runBridgeLoop` in `src/bridge/bridgeMain.ts:141` is invoked by the dedicated `claude bridge` subcommand — a different entrypoint that does not share `setup()`.

See task #10 for the bridge/remote-control deep dive.

---

## 8. Cron / Scheduled Prompts

`src/hooks/useScheduledTasks.ts:40` hosts the scheduler inside the REPL. Mounted unconditionally; runtime gate `isKairosCronEnabled()` short-circuits in the `useEffect`.

- Core scheduler lives in `src/utils/cronScheduler.ts` (`createCronScheduler`) so SDK/`-p` mode can reuse it (`src/cli/print.ts`).
- Fired tasks enqueue to the command queue at `'later'` priority — REPL drains them between turns via `useCommandQueue`.
- Cron state persisted via `src/utils/cronTasks.ts`; lock via `src/utils/cronTasksLock.ts`.
- `isLoading` is read through a ref so the scheduler's getter doesn't capture stale closures.

Assistant mode (`#20425`) no longer forces `--proactive`; `isLoading` now drops between turns, making the `assistantMode` bypass a latency nicety rather than a starvation fix.

---

## 9. Cleanup Registry

`src/utils/cleanupRegistry.ts` (`registerCleanup`) is the universal disposal mechanism. Every long-lived peer registers a cleanup callback:

- `fileChangedWatcher` dispose — closes chokidar.
- `preventSleep` cleanupRegistered flag — kills caffeinate.
- `SessionsWebSocket` — closes the socket.
- `LocalShellTask` / `LocalAgentTask` — kills spawned children.
- `concurrentSessions` — removes PID file.
- UDS messaging server — unlinks the socket.

Shutdown path runs these (see task #7 for the full sequence).

---

## 10. Per-Turn Transient Concurrency

Not strictly "alongside the REPL" since they're driven by user queries, but they run in parallel once spawned:

- **BashTool** → `src/utils/Shell.ts` / `ShellCommand.ts` — spawns per-command subprocess, Pipes stdout/stderr back via streams.
- **LocalShellTask** → `src/tasks/LocalShellTask/LocalShellTask.tsx` — background shell jobs.
- **LocalAgentTask** → `src/tasks/LocalAgentTask/LocalAgentTask.tsx` — spawns `claude` child process for sub-agent work.
- **AgentTool** → `src/tools/AgentTool/AgentTool.tsx` — launches a subagent query (can be in-process teammate via `InProcessTeammateTask`).
- **SendMessageTool** / **BriefTool** — post messages to teammates (interprocess via UDS or intraprocess via signal).
- **ScheduleCronTool / CronCreateTool** — register scheduled tasks.

See task #14 (tools) for a tool-by-tool deep dive.

---

## 11. Idle Notification Timer

`src/screens/REPL.tsx:3910-3940` sets a `setTimeout` each render that fires `sendNotification({message: 'Claude is waiting for your input', notificationType: 'idle_prompt'})` if the user has been idle past `messageIdleNotifThresholdMs` AND a query has completed AND there's no active dialog/tool UI. Re-armed on every effect run, cleared on re-render.

---

## 12. Control Flow Summary

```
main.tsx action():
  preAction → init()             [analytics meter, mTLS, proxies, graceful shutdown, upstream proxy]
  setup()                        [stage 2 §2 above]
  ↓ promise join: setup + getCommands + getAgents
  showSetupScreens (trust, onboarding)
  renderAndRun(root, <App/>):
    root.render(element)
    startDeferredPrefetches()    [§3]
    await root.waitUntilExit()
      ↓ REPL mount
      REPL.tsx effects:
        - useRemoteSession / useDirectConnect / useSSHSession [§7]
        - useReplBridge                                      [§7]
        - useScheduledTasks (cron)                           [§8]
        - MCPConnectionManager (via children tree)            [§6]
        - after submitCount === 1 → startBackgroundHousekeeping()  [§4]
        - idle notification timer                             [§11]
    gracefulShutdown(0)           [task #7]
```

For the `-p` / headless path (`src/main.tsx:2810+`):
- Same `setup()` work.
- No `root.render`; `runHeadless` runs to completion in `src/cli/print.ts`.
- `startDeferredPrefetches()` + `startBackgroundHousekeeping()` fire inline (no typing window to hide the cost).

---

## 13. Integration Points

- **`setup()` / `init()`** — see tasks #1 (setup) and #11 (auth/init). Those set the stage; this task catalogs what *stays running*.
- **REPL (`src/screens/REPL.tsx`)** — the mount point for bridges, cron, housekeeping trigger. See task #9.
- **`query.ts`** — the per-turn async pattern. See task #17. `preventSleep` is driven by query lifecycle.
- **`print.ts`** — headless path; reuses the same concurrent primitives. See task #12.
- **Hooks system** — `fileChangedWatcher`, `skillChangeDetector`, `settingsChangeDetector`, `loadPluginHooks + setupPluginHookHotReload` all depend on the hooks infrastructure.
- **MCP, bridge, remote, direct-connect, ssh** — tasks #2 and #10 own the protocol layers; this task surveys their concurrency model.
- **Cron & Tools** — tasks #14 (tools) and the dedicated cron deep-dive (follow-up).
- **Analytics / logging** — `sinks.ts`, `initializeAnalyticsSink`, `initializeErrorLogSink`, datadog/firstParty emitters. See task #16 for STATE+config context.

---

## 14. Recommendations for Follow-Up Analysis

1. **Daemon architecture (flagged, already task #8)** — `SessionKind` includes `'daemon' | 'daemon-worker'`, and the assistant mode (`--assistant`) spins up a daemon subprocess (`src/main.tsx:3288-3290`). Bridge/remote plus daemon warrant a lifecycle diagram: supervisor, worker, capacity wake (`src/bridge/capacityWake.ts`), heartbeat.
2. **Cron scheduler internals** — `src/utils/cronScheduler.ts`, `cronTasksLock.ts`, `cronJitterConfig.ts`. Needs a deep dive on persistence format, jitter config, kill-switch semantics (`isKilled` polling).
3. **MCP connection pool internals (complement to task #2)** — `client.ts` 3348 LOC. Reconnection/backoff, auth flows, elicitation handler, official registry lookup, channel allowlist/permissions are independent subsystems.
4. **File/settings/skill detectors unified view** — three chokidar watchers plus MDM poll. A combined "watcher registry" analysis would help reason about stale caches after external edits.
5. **Cleanup ordering** — `cleanupRegistry.ts` runs callbacks in registration order; any new entry should document its ordering expectations. Follow-up overlaps task #7.
6. **concurrentSessions.ts** — the session-registry file semantics (kind, status, lock files) deserve a standalone doc: who writes/reads, resume semantics across crashes, PID reclamation.
7. **Event loop stall detector + sdkHeapDumpMonitor** — ant-only observability stack. Low priority but worth mapping if task #16 touches telemetry.
8. **UDS messaging server** — `utils/udsMessaging.js` (not under `src/utils` in this snapshot — may be in a separate bundle). Worth cross-referencing with task #10 (bridge/remote) and task #5 (interrupts).
9. **Bridge/remote WebSocket peers** — already scheduled as task #10; flag the dedicated React hooks (`useReplBridge`, `useRemoteSession`, `useDirectConnect`, `useSSHSession`) as the REPL-side integration surface.

---

## 15. Open Questions / Ambiguities

- `startBackgroundHousekeeping()` mounts at `submitCount === 1` — what happens for sessions that never submit (immediate Ctrl+C)? The housekeeping never runs, but the pre-render and deferred prefetches already did. Is that intentional?
- `startDeferredPrefetches()` fires in headless mode but comment warns housekeeping is "bookkeeping the next interactive session reconciles" — yet headless **also** calls `startBackgroundHousekeeping()` at `src/main.tsx:2818`. Is the comment stale?
- `preventSleep` ref counting: is there any code path that `start`s without a matching `stop` (e.g., aborted query, error before finally)? Would cause permanent caffeinate until process exit.
- `settingsChangeDetector` + `fileChangedWatcher` + `skillChangeDetector` are three independent chokidar watchers. Any overlap in watched paths? Chokidar does not de-dupe across instances.
- UDS messaging server is awaited inside `setup()`; if it fails, does `setup()` crash the whole process or degrade gracefully?
- `eventLoopStallDetector` + `sdkHeapDumpMonitor` are `"external" === 'ant'` gated — confirm these strings are DCE'd in external builds (same gate style as `setup.ts:295-300` contextCollapse).
