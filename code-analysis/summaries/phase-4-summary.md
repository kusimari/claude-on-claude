# Phase 4 Summary — Daemon, Bridge, and Templates

Synthesized from two Phase 4 reports:
- `sub-analysis/wave-1/phase-4/16-daemon-mode.md`
- `sub-analysis/wave-1/phase-4/17-bridge-and-templates.md`

---

## Report 16 — Daemon Architecture and Operation

### Key Findings

- **Daemon source is excluded from the extracted tree.** Both `src/daemon/main.ts` and `src/daemon/workerRegistry.ts` are referenced by `cli.tsx:100-106` and `cli.tsx:164-180` but the directory doesn't exist — gated out by `feature('DAEMON')` at build. Only the *contract* with the daemon is analyzable.
- **Daemon = long-running supervisor + lean children.** `claude daemon` supervises workers spawned as `claude --daemon-worker <kind>`. Workers skip `enableConfigs()`/`initSinks()` at dispatch (perf-critical) and opt in themselves; supervisor runs both inline.
- **Shared auth via IPC.** Supervisor owns the `AuthManager`; workers request tokens through `opts.getAccessToken()` / `opts.onAuth401()` callbacks rather than reading OAuth state directly, avoiding lock contention and duplicated refreshes.
- **Error taxonomy is load-bearing.** `BridgeHeadlessPermanentError` → `EXIT_CODE_PERMANENT` → supervisor "parks" the worker (no respawn). Plain `Error` → transient → respawn with backoff. Classification is spread across preflight checks and a runtime `fatalExit` flag.
- **At least three worker kinds exist**: `remoteControl` (bridge, visible), `cron` (inferred from `cronScheduler.ts:123-124`), `assistant` (Kairos/Agent-SDK, inferred from `main.tsx:1052-1056`). Each has different init/entitlement requirements.

### Critical Code Locations

- `src/entrypoints/cli.tsx:95-106` — worker fast-path dispatch (pre-config)
- `src/entrypoints/cli.tsx:164-180` — supervisor fast-path dispatch
- `src/bridge/bridgeMain.ts:2770-2965` — `runBridgeHeadless` (only analyzable worker body)
- `src/bridge/bridgeMain.ts:2778-2783` — `BridgeHeadlessPermanentError` taxonomy contract
- `src/bridge/bridgeMain.ts:2785-2797` — `HeadlessBridgeOpts` IPC contract
- `src/bridge/bridgeMain.ts:279-313` — v2 daemon session token refresh (CC-1263)
- `src/utils/concurrentSessions.ts:18,27-37` — `SessionKind` enum, `CLAUDE_CODE_SESSION_KIND` env override
- `src/utils/cronScheduler.ts:75-124` — `assistantMode`, `onFireTask`, permanent-task partition
- `src/main.tsx:1052-1056,3842` — hidden `--assistant` flag (skip Kairos recheck)

### Integration Points

- **`daemon.json`** — supervisor config (schema not extracted)
- **`~/.claude/sessions/<pid>.json`** — PID registry tagged `daemon`/`daemon-worker` for `claude ps`
- **`scheduled_tasks.json`** — partitioned by `permanent` flag between daemon and non-daemon
- **Bridge API** — `registerBridgeEnvironment`, `reconnectSession` (shared with interactive bridge)
- **IPC channel** — token hand-off, 401 refresh, log forwarding, abort signal (implementation missing)
- **`CLAUDE_CODE_MESSAGING_SOCKET`** — UDS inbox, possible supervisor↔worker IPC
- **Upstream proxy** (`upstreamproxy.ts:13,139`) — designed around "supervisor restart retry"

### Recommendations

1. Re-extract with `feature('DAEMON')` and `feature('KAIROS')` enabled — supervisor internals and assistant module are gated out.
2. Dedicated follow-up tasks for: `daemonMain` lifecycle (backoff/shutdown), `workerRegistry` policy layer, daemon `AuthManager` IPC, full worker-kind enumeration.
3. Flag the WSL pruning gap (`concurrentSessions.ts:194-201`) — stale daemon-worker PID files can accumulate cross-platform.

### Complexity: **Complex** — and only partially analyzable

Worker management is non-trivial: bespoke error taxonomy, cross-process auth callbacks, three+ worker kinds with divergent init, state duplication with `--bg` sessions, and a v1/v2 token-refresh split tied to specific past incidents (CC-1263). Most of the complexity lives in the missing `daemon/` source.

---

## Report 17 — Bridge / Remote-Control and Template Jobs

### Key Findings

- **Three front doors converge on one poll loop.** Standalone (`claude remote-control`), REPL-hosted (`/remote-control`), and daemon-worker (`runBridgeHeadless`) all end up in `runBridgeLoop` (env-based) or the v2 SSE+CCR transport (env-less). Split is about which code builds the config, not the loop.
- **Bridge module is ~12.6k LOC across 31 files** — `bridgeMain.ts` (2999 LOC) + `replBridge.ts` (2406 LOC) are the heavyweights; `remoteBridgeCore.ts` (1008 LOC) is the env-less variant.
- **Init-bypass is deliberate.** All bridge entrypoints skip `init.ts` and re-do only the narrow subset they need (`enableConfigs`, `initSinks`, CWD, trust check) to keep bridge cold-start fast. `bridgeMain` must call `process.exit(0)` explicitly (line 2767) because nothing else cleans up.
- **v1 env-based vs v2 env-less split.** v1 uses the Environments API (register/poll/ack/heartbeat) with WS or SSE transport; v2 is SSE+CCR only, skips Environments, uses `/bridge` endpoint returning `worker_jwt`+`worker_epoch`. v2 is REPL-only today — daemon, standalone, and print still take v1.
- **Templates subsystem is MISSING from snapshot.** `cli.tsx:211-222` dispatches `claude new|list|reply` to `src/cli/handlers/templateJobs.js`, but that file does not exist in the extracted tree. `feature('TEMPLATES')` gates it out of external builds.

### Critical Code Locations

- `src/entrypoints/cli.tsx:100-106,108-162,211-222` — bridge/daemon-worker/templates fast-paths
- `src/bridge/bridgeMain.ts:141` — `runBridgeLoop` (~1400 lines)
- `src/bridge/bridgeMain.ts:650-730` — at-capacity heartbeat mode
- `src/bridge/bridgeMain.ts:859-1204` — session spawn path
- `src/bridge/bridgeMain.ts:1403-1579` — graceful shutdown + resumable-shutdown short-circuit
- `src/bridge/bridgeMain.ts:1980-2435` — pre-loop setup (parseArgs, OAuth, trust, spawn-mode dialog)
- `src/bridge/bridgeMain.ts:2810-2965` — `runBridgeHeadless`
- `src/bridge/initReplBridge.ts:110-240` — phased gates + dead-token cross-process backoff
- `src/bridge/initReplBridge.ts:258-378` — title derivation (Haiku-generated)
- `src/bridge/initReplBridge.ts:410-452` — v1/v2 env-less branch
- `src/bridge/replBridge.ts:91-221,260,381` — `BridgeCoreParams`, `initBridgeCore`, `tryReconnectInPlace`
- `src/bridge/remoteBridgeCore.ts:140` — `initEnvLessBridgeCore`
- Missing: `src/cli/handlers/templateJobs.js`

### Integration Points

- **Auth** (`utils/auth.ts`) — `getClaudeAIOAuthTokens`, proactive refresh before any bridge call; REPL adds cross-process dead-token backoff via global config keys `bridgeOauthDeadExpiresAt`/`bridgeOauthDeadFailCount`.
- **Config** (`utils/config.ts`) — `enableConfigs()` mandatory; project-level `remoteControlSpawnMode` persists same-dir-vs-worktree choice.
- **Policy** (`services/policyLimits`) — MDM `allow_remote_control` flag; blocking load in standalone path.
- **Worktree** (`utils/worktree.ts`) — `createAgentWorktree`/`removeAgentWorktree` for spawn-per-worktree.
- **Session-ingress** (`utils/sessionIngressAuth.ts`) — JWT refresh propagation to WS readers.
- **Concurrent sessions** (`utils/concurrentSessions.ts`) — `updateSessionBridgeId` exposes bridge session to `ps`/`attach`.
- **Cleanup registry** (`utils/cleanupRegistry.ts`) — teardown hooks for graceful shutdown.
- **Commands registry avoidance** — `replBridge.ts` + `remoteBridgeCore.ts` deliberately do NOT import `src/commands.ts`; callbacks are injected via `BridgeCoreParams` to keep Agent SDK bundle small.

### Recommendations

1. Session-ingress protocol deep-dive (`replBridgeTransport.ts` + `bridgeMessaging.ts`): v1 WS vs v2 SSE+CCR wire format, sequence numbering, reconnection.
2. Token-refresh scheduler internals (`jwtUtils.ts`): three different refresh flows with subtle laptop-wake races.
3. `sessionRunner.ts` child-process shape: stdout parsing for activity events, kill-tree semantics, `onFirstUserMessage` closure.
4. `FlushGate` + initial-history ordering correctness.
5. Cross-reference with daemon task — `runBridgeHeadless` is the only worker body we can analyze.
6. Templates system — await `templateJobs.js` appearance or source from internal build.

### Complexity: **Complex**

Huge surface area (12.6k LOC, 31 files), three entry paths, two protocol generations (v1/v2) that coexist, bespoke retry taxonomies, cross-process backoff state, CCR shim for `session_*`↔`cse_*` id retagging at ~30 call sites, and subtle invariants (sleep detection via 2× backoff threshold, `pendingCleanups` discipline, flush-gate ordering).

---

## Cross-Report Observations

- **Daemon and bridge are deeply entangled.** `runBridgeHeadless` is the only `remoteControl` worker kind; it's also the only daemon worker body visible. The daemon's `AuthManager` IPC shape is defined *by* the bridge's `HeadlessBridgeOpts` contract.
- **Two large subsystems stripped from snapshot**: `src/daemon/` (entire directory) and `src/cli/handlers/templateJobs.js`. Both feature-gated. Analysis of supervisor lifecycle, worker dispatch policy, and template fleet view is limited to call-site comments.
- **Consistent init-bypass philosophy** across bridge, daemon supervisor, and daemon workers — each re-does the narrow subset of `init.ts` it actually needs, trading duplication for fast startup and minimal bundle weight.
