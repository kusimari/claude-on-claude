# Phase 3 Summary — Background, Interrupts, Persistence, Cleanup

Covers reports 12–15: concurrent processes, interruption/error recovery, session persistence, shutdown sequence.

---

## 12 — Concurrent Processes / Background Activity

**Complexity**: Complex

### Key findings
- Six tiers of concurrent activity alongside REPL: (1) pre-render prefetches from `setup()`, (2) deferred prefetches post-render, (3) `startBackgroundHousekeeping` gated on `submitCount === 1`, (4) long-lived peers (sinks, preventSleep, detectors), (5) per-turn transient tools, (6) external peers (MCP, bridge, cron, UDS).
- All single event-loop — "concurrent" = overlapping async; heavy CPU in subprocesses (analytics, MCP servers, Bash) or `.unref()`'d intervals.
- **`.unref()` discipline universal**: every timer/interval unref'd so nothing holds the process alive; housekeeping reschedules if `getLastInteractionTime()` within 60s (admission control against slow I/O).
- **Three chokidar watchers**: `fileChangedWatcher`, `settingsChangeDetector`, `skillChangeDetector` — independent instances; chokidar doesn't de-dupe across instances (potential overlap).
- **Bridge/Remote/Direct/SSH**: mutually-exclusive transport selection via `activeRemote = sshRemote ?? directConnect ?? remoteSession`; only one receives messages.

### Critical code locations
- `src/setup.ts:95-380` pre-render prefetches (UDS, watchers, hooks, sinks, API key).
- `src/main.tsx:388-431` `startDeferredPrefetches` (user context, system context, Statsig, MCP URLs, gates, model caps).
- `src/utils/backgroundHousekeeping.ts:31-94` — MagicDocs, skill improvement, extract memories, autoDream, plugin auto-update, very-slow-ops scheduler.
- `src/screens/REPL.tsx:3903-3907` housekeeping mount; `:3910-3940` idle notification timer; `:1421-1422` activeRemote selection.
- `src/services/preventSleep.ts:36-58` macOS ref-counted caffeinate subprocess.
- `src/utils/settings/changeDetector.ts` — `MDM_POLL_INTERVAL_MS=30min`, `INTERNAL_WRITE_WINDOW_MS=5s` dedup, `DELETION_GRACE_MS` for auto-update delete+recreate.
- `src/utils/concurrentSessions.ts` — PID file + `SessionKind` (`interactive`/`bg`/`daemon`/`daemon-worker`).
- `src/hooks/useScheduledTasks.ts:40` cron mount; `src/utils/cronScheduler.ts` `createCronScheduler`.

### Integration points
- **Cleanup registry** (`src/utils/cleanupRegistry.ts`): universal disposal; every long-lived peer registers a callback.
- **MCP**: `useManageMCPConnections` spawns subprocesses (stdio) or remote (HTTP/SSE/WS); `claude.ai` connector bounded wait (`CLAUDE_AI_MCP_TIMEOUT_MS`).
- **Cron**: fired tasks enqueue at `'later'` priority; REPL drains between turns via `useCommandQueue`.
- **Hooks**: `fileChangedWatcher` + `skillChangeDetector` + `settingsChangeDetector` + `loadPluginHooks` + hot-reload all depend on hooks infrastructure.

### Recommendations
- Daemon architecture deep-dive (covered by Phase 4 report 17).
- Cron scheduler internals — persistence format, jitter config, kill-switch.
- Unified watcher registry — three independent chokidar instances.
- `eventLoopStallDetector` + `sdkHeapDumpMonitor` ant-only observability.

### Open questions
- `startBackgroundHousekeeping` on `submitCount === 1` — what happens for sessions that never submit?
- Headless path calls both `startDeferredPrefetches` and `startBackgroundHousekeeping` — comment about "next interactive session reconciles" may be stale.
- `preventSleep` ref-count leak risk if `start` without matching `stop` on error path.

---

## 13 — Interruption Handling & Error Recovery

**Complexity**: Very Complex

### Key findings
- **Three non-overlapping layers**: (1) process-level signals (`gracefulShutdown.ts`), (2) in-process `AbortController` tree, (3) API retry/fallback (`withRetry.ts`). Signal → controller → retry cascade.
- Signal handlers: SIGINT (exit 0, skipped in print mode), SIGTERM (143), SIGHUP (129, non-Windows), `uncaughtException` + `unhandledRejection` logged but not fatal. All funnel to `void gracefulShutdown()`.
- **Bun sigaction pin** (gracefulShutdown.ts:254): `onExit(() => {})` no-op subscriber prevents signal-exit v4's `removeListener` from resetting kernel sigactions mid-session (silent SIGTERM no-op bug).
- **Orphan detection** (gracefulShutdown.ts:281-296): 30s `.unref()`'d interval polls `stdout.writable && stdin.readable` because macOS revokes TTY fds without SIGHUP on force-close.
- **Abort reasons semantic**: `'user-cancel'` (Esc/Ctrl+C), `'background'` (Ctrl+B), `'interrupt'` ('now'-priority queued). Dispatched via `signal.reason` check.
- `withRetry.ts` (822 LOC): foreground/background 529 triage (`FOREGROUND_529_RETRY_SOURCES`), persistent-mode heartbeating (`CLAUDE_CODE_UNATTENDED_RETRY`, 30s yields), fast-mode cooldown, Opus → fallback chain, provider-specific auth recovery.

### Critical code locations
- `src/utils/gracefulShutdown.ts:237-334` `setupGracefulShutdown`; `:254` Bun pin; `:281-296` orphan detect; `:391-523` pipeline; `:193-232` forceExit + SIGKILL fallback.
- `src/utils/abortController.ts:16-22` `createAbortController` (maxListeners 50); `:68-99` `createChildAbortController` with WeakRef.
- `src/utils/combinedAbortSignal.ts:15-47` plain setTimeout (avoids Bun `AbortSignal.timeout` 2.4KB leak).
- `src/utils/errors.ts:27-33` `isAbortError` — checks `AbortError | APIUserAbortError | name === 'AbortError'` (SDK class names minified).
- `src/screens/REPL.tsx:865-869` controller state + ref mirror; `:2125-2163` `onCancel`; `:2528` Ctrl+B background; `:4102` 'now'-priority interrupt.
- `src/hooks/useCancelRequest.ts:87-115` priority order; `:225-273` two-press kill-agents.
- `src/services/api/withRetry.ts:170-517` main loop; `:62-82` FOREGROUND_529_RETRY_SOURCES; `:477-506` persistent heartbeat; `:267-314` fast-mode; `:326-365` Opus fallback; `:550-595` context-overflow self-heal; `:631-694` Bedrock/Vertex recovery.
- `src/entrypoints/init.ts:215-236` ConfigParseError recovery (invalid-config dialog).

### Integration points
- **Graceful-shutdown ↔ hooks**: `executeSessionEndHooks` budget (`CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`, default 1.5s) resolved before failsafe arms.
- **AbortController ↔ tool execution**: every tool receives `toolUseContext.abortSignal`; subprocesses responsible for propagation.
- **withRetry ↔ signal**: loop header checks `signal?.aborted`; `sleep(delayMs, signal, { abortError })` aborts mid-sleep.
- **withRetry ↔ fastMode**: separate `utils/fastMode.ts` holds cooldown state; retry triggers via `triggerFastModeCooldown`.
- **User interrupt ↔ permission UI**: `onCancel` calls `toolUseConfirmQueue[0]?.onAbort()`.
- **Remote mode**: `activeRemote.cancelRequest()` routes through CCR socket, not local abort.

### Recommendations
- Deep dive on `rateLimitMessages.ts` + `SystemAPIErrorMessage` UI rendering.
- Tool-level cancellation (`ShellCommand.ts`, computerUse) — subprocess tree killing.
- Permission prompt `onAbort` lifecycle.
- Session backgrounding (`useSessionBackgrounding`, Ctrl+B handoff).
- Persistent-retry keep-alive dedicated channel (ANT-344 TODO).

### Invariants
- Terminal restoration sync-first, before any await.
- `forceExit` SIGKILL fallback on EIO (dead TTY after SIGHUP/SSH disconnect).
- Background 529 retry dropped — foreground-only to prevent cascade amplification.
- `FallbackTriggeredError` after `MAX_529_RETRIES=3` bubbles to query loop (report 11).

---

## 14 — Session Persistence

**Complexity**: Very Complex (flagged "most carefully-engineered subsystem examined")

### Key findings
- JSONL append-only log per session at `~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl` (files 0o600, dirs 0o700). 18+ `Entry` types discriminated in `Project.appendEntry` (sessionStorage.ts:1128-1265).
- **Write pipeline**: per-file batched queue, `FLUSH_INTERVAL_MS=100` (10 for CCR v2/ingress), `MAX_CHUNK_BYTES=100MB` cap. `pendingEntries` buffer pre-materialization so metadata-only orphan files don't leak.
- **64KB tail invariant**: pickers read only last 64KB of each session file (`LITE_READ_BUF_SIZE`); `reAppendSessionMetadata` (sessionStorage.ts:721-839) re-writes `customTitle`/`tag`/`gitBranch`/PR/agent fields at exit + on resume so they stay in the window. `ai-title` **intentionally never re-appended** to avoid clobbering user renames.
- **Load pipeline**: `SKIP_PRECOMPACT_THRESHOLD=5MB` threshold — small files parseJSONL; large files `readTranscriptForLoad` chunked forward scan with attr-snap fd-level skip + boundary truncate. `walkChainBeforeParse` (sessionStorage.ts:3306-3466) dead-fork elimination relies on `{"parentUuid":` as first-key invariant.
- **Three persistence paths**: (a) local JSONL, (b) v1 Session Ingress (`persistToRemote` → `appendSessionLog`), (c) CCR v2 internal events (`setInternalEventWriter`). Plus prompt history (`~/.claude/history.jsonl`, global, locked).

### Critical code locations
- `src/utils/sessionStorage.ts:440-467` `Project` singleton + cleanup register; `:618-686` write queue drain; `:993-1083` `insertMessageChain` parentUuid stamping; `:976-991` `materializeSessionFile`; `:721-839` `reAppendSessionMetadata`; `:871-951` `removeMessageByUuid` tombstone (64KB fast path, 50MB slow path cap).
- `:1530-1534` `adoptResumedSessionFile`; `:1587-1622` `hydrateRemoteSession`; `:1632-1723` `hydrateFromCCRv2InternalEvents`.
- `:3472-3813` `loadTranscriptFile`; `:3306-3466` `walkChainBeforeParse`; `:3157-3224` `scanPreBoundaryMetadata`; `:2069-2094` `buildConversationChain`; `:2224-2243` `checkResumeConsistency`.
- `:1839-1956` `applyPreservedSegmentRelinks`; `:1982-2039` `applySnipRemovals`.
- `src/utils/sessionStoragePortable.ts:311-319` `sanitizePath`; `:403-466` `resolveSessionFilePath`; `:717-793` `readTranscriptForLoad`.
- `src/utils/conversationRecovery.ts:456-597` `loadConversationForResume`; `src/utils/sessionRestore.ts:409-551` `processResumedConversation`.
- `src/history.ts:292-327` `immediateFlushHistory` with `lock(historyPath)`; `:453-464` `removeLastFromHistory` skippedTimestamps.

### Integration points
- **Bootstrap state**: atomic `switchSession(sid, projectDir)` (CC-34 fix for transcript_path mismatch); `getSessionId`, `getSessionProjectDir`, `isSessionPersistenceDisabled`.
- **Cleanup registry**: exit handler flushes queue + re-appends metadata (sessionStorage.ts:448-462).
- **Worktree**: `worktree-state` entry tri-state (`undefined`/`null`/object); `restoreWorktreeForResume` cd's on resume.
- **Compaction**: `compact_boundary` + `preservedSegment` writes; `applyPreservedSegmentRelinks` in-memory splice zeroes usage tokens (prevents auto-compact spiral on resume).
- **Snip**: `snipMetadata.removedUuids` drives `applySnipRemovals` path-compression.
- **Context-collapse**: `marble-origami-commit` ordered log + `marble-origami-snapshot` last-wins.
- **Remote ingress**: failure → `gracefulShutdownSync(1, 'other')` — fatal for managed deployments; epoch mismatch 409 re-thrown to avoid race with graceful shutdown.
- **Resume modes**: `--continue`/`--resume <id>`/`--resume <path>`/`--resume <title>`/`--fork-session`; fork keeps fresh sessionId, seeds `recordContentReplacement`, strips `worktreeSession` to avoid double-ownership.

### Recommendations
- Compaction-stack deep-dive (`compact.ts`, `reactiveCompact.ts`) — tightly coupled to boundary handling.
- `deserializeMessagesWithInterruptDetection` legacy migrations + interrupt detection.
- Content-replacement storage (`toolResultStorage.ts`) FROZEN classification on resume.
- File history (`fileHistory.ts`) snapshot creation + `isSnapshotUpdate` chain.
- Remote Session Ingress protocol + CCR v2 internal-event client.
- Resume picker UI (`ResumeConversation.tsx`) progressive enrichment consumption.

### Invariants (load-bearing)
- Session ID + projectDir atomicity (CC-34).
- `parentUuid` first key in JSON serialization (walkChainBeforeParse relies on `{"parentUuid":` prefix).
- Compact boundary: `parentUuid = null`, `logicalParentUuid = originalParent`.
- Session-stamp re-stamping on fork/resume (spread-then-override ordering).
- Agent sidechain writes bypass main dedup set.
- `ai-title` never re-appended (customTitle takes precedence).
- Zero usage on preserved segment messages (prevents auto-compact spiral).
- Lock on history.jsonl (global); no lock on transcripts (per-session, no contention).
- Transcripts written with `{mode: 0o600}`; dirs `0o700`.

---

## 15 — Cleanup and Exit

**Complexity**: Very Complex

### Key findings
- Single centralized 13-step exit pipeline (`src/utils/gracefulShutdown.ts`, 529 LOC) + flat unordered parallel `cleanupRegistry` `Set<() => Promise<void>>` (25 LOC).
- **Sync-first ordering**: `cleanupTerminalModes()` + `printResumeHint()` use `writeSync` **before any `await`** so terminal state is sane + resume hint visible even under SIGKILL mid-cleanup.
- **Time-bounded phases**: registry 2000ms, SessionEnd hooks default 1500ms (env-overridable), MCP stdio SIGINT→SIGTERM 100ms→SIGKILL 500ms→absolute 600ms, analytics flush 500ms, global failsafe `max(5000, hookBudget + 3500)`.
- **`forceExit` SIGKILL fallback**: if `process.exit()` throws EIO (Bun flushing stdout to dead fd on SIGHUP/SSH-disconnect) → `process.kill(pid, 'SIGKILL')`.
- **43 registering files**: transports (MCP, LSP, bridge, tmux, shell tasks, swarm), persistence (session, config, history, activity), watchers/locks (settings, skills, keybindings, cron, CU, team memory, git-fs), output (OTEL, Perfetto, asciicast, NDJSON guard, terminal panel), install (native, plugins, proxy).
- Exit codes: 0 (normal), 1 (~80 validation/auth/trust-rejection sites), 129 (SIGHUP + orphan), 143 (SIGTERM). `ExitReason` enum: `clear`/`resume`/`logout`/`prompt_input_exit`/`other`/`bypass_permissions_disabled` propagated to SessionEnd hooks.

### Critical code locations
- `src/utils/gracefulShutdown.ts:391-523` pipeline; `:336-359` `gracefulShutdownSync`; `:193-232` `forceExit`; `:59-136` `cleanupTerminalModes` (internal ordering 11 steps); `:144-184` `printResumeHint`; `:237-334` `setupGracefulShutdown`; `:254` Bun signal-exit v4 pin.
- `src/utils/cleanupRegistry.ts:23-25` `runCleanupFunctions` via `Promise.all`.
- `src/services/mcp/client.ts:1429-1562` stdio SIGINT→SIGTERM→SIGKILL escalation (100+ LOC inner state machine; 600ms absolute failsafe; orphan-then-let-OS-reap).
- `src/cli/print.ts:1027-1034` print-mode SIGINT handler (overrides global); `:971-973` runHeadless exit.
- `src/utils/sessionStorage.ts:448-462` flush + re-append metadata cleanup.
- `src/entrypoints/sdk/coreSchemas.ts:747-754` `EXIT_REASONS` enum.
- `src/utils/hooks.ts:4097-4141` `executeSessionEndHooks`; `:176-182` `getSessionEndHookTimeoutMs`.

### Integration points
- **Session persistence**: final `project?.flush() + reAppendSessionMetadata()` idempotent.
- **MCP**: only subsystem doing signal escalation in cleanup.
- **Hooks**: SessionEnd budget propagated to both `AbortSignal.timeout` and `timeoutMs` option; failures write to stderr (Ink already unmounted).
- **Analytics**: `shutdown1PEventLogging` + `shutdownDatadog` + OTEL `shutdownTelemetry`; 500ms cap protects against slow networks.
- **Interruption handling**: SIGINT → `gracefulShutdown(0)` non-print; print mode handles own SIGINT → abort query → shutdown.
- **Ink**: `Ink#unmount()` hooked via `onExit` signal-exit; `detachForShutdown()` handshake prevents double-unmount and DECRC cursor-reset (which would clobber resume hint).

### Recommendations
- Verify daemon-mode shutdown uses same pipeline (Phase 4 report).
- Document bridge/MCP transport shutdown race potential (both register cleanup, both use transports).
- Resume hint correctness: printed before flush — flush-failure → user told to `--resume` but file may be inconsistent (intentional trade-off).
- Analytics 500ms budget can drop last ~500ms of events on slow networks (acknowledged).
- Test matrix: double-call idempotency, failsafe-fired path, EIO → SIGKILL, print-mode SIGINT skip.

### Invariants (critical ordering)
- Sync terminal reset + resume hint **before any await**.
- `DISABLE_MOUSE_TRACKING` before Ink unmount (round-trip to stop events).
- Exit alt-screen before `printResumeHint` (else hint lands on alt buffer, lost).
- Ink unmount **through** its own mechanism (not manual `writeSync(1049l)` — would DECRC twice, cursor wrong).
- `detachForShutdown()` after unmount (prevents deferred signal-exit double-unmount clobber on tmux).
- Session flush + registry before SessionEnd hooks (user hooks may hang; session data persisted first).
- `reAppendSessionMetadata()` after flush (64KB tail window).
- `profileReport` before analytics shutdown (analytics cancels profile timers).
- `tengu_cache_eviction_hint` before analytics flush (event must reach pipeline).
- Failsafe budget **after** hook budget resolved (gh-32712 fix; else user's 10s hook budget silently truncated to 5s).
- `forceExit()` always last (failsafe timer clears on entry).
- Cleanup registry set is `Set`, registration order **not** preserved — subsystems encode ordering internally (see MCP stdio).

---

## Cross-cutting themes (Phase 3)

- **Time-bounded everywhere**: every async phase in shutdown has a cap (2s/1.5s/500ms/600ms); global failsafe `max(5000, hookBudget + 3500)`. Nothing can hang the process.
- **Sync-first for user-visible state**: terminal restoration + resume hint use `writeSync` before first `await` — visible even under SIGKILL.
- **`.unref()` discipline**: every background timer/interval unref'd so it can't hold the process alive.
- **Three-layer cancellation**: OS signals → AbortController tree (WeakRef for memory safety) → API retry loop; each layer has distinct semantics + domain-specific handling.
- **Semantic abort reasons**: `'user-cancel'` / `'background'` / `'interrupt'` dispatched via `signal.reason` for differential handling (abandon vs handoff vs requeue).
- **Platform-specific workarounds**: Bun signal-exit v4 pin (kernel sigaction preservation), Bun `AbortSignal.timeout` native-memory bug, macOS TTY orphan detect, EIO→SIGKILL fallback.
- **64KB tail window discipline**: readLiteMetadata + scanPreBoundaryMetadata + reAppendSessionMetadata all enforce the same tail-window invariant for picker visibility.
- **Append-only log with in-memory filters**: preservedSegment splice + snip removedUuids relink handle logical deletion without rewriting JSONL; tombstones the exception (OOM-capped at 50MB).
- **Parallel cleanup, ordering via phases**: registry is unordered parallel `Promise.all`; global ordering enforced by pipeline phase structure, not registration order.
- **Provider-agnostic retry semantics**: `withRetry` handles 429/529/401/403/connection errors uniformly; provider-specific auth recovery (Bedrock, Vertex, OAuth, API key) plugged in via clear-cache callbacks.
- **Session-persistence is the most carefully-engineered subsystem** observed — 5+ persistence paths, 5MB threshold load split, tombstone fast/slow paths, 64KB window invariants, 9+ documented post-incident fixes.
- **Post-incident fingerprints pervasive**: Phase 3 code references CC-34, #22453, #14373, #23537, adamr-20260320, inc-3930, inc-3694, inc-4718, gh-32712, ANT-344 — each invariant is a scar.
