# 17 — Bridge / Remote-Control and Template Jobs

**Scope.** The "Remote Control" subsystem (aka "bridge") lets a local
`claude` process expose itself as a remotely controllable work
environment for claude.ai/code or the Claude mobile app. Two delivery
shapes coexist: (a) a dedicated standalone binary invocation
(`claude remote-control …`) that runs a poll loop and spawns child
sessions on demand, and (b) a REPL-hosted bridge that piggybacks on a
live interactive session. This analysis walks both paths end-to-end,
covers the daemon-worker form (`runBridgeHeadless`), catalogs the
support modules in `src/bridge/`, and flags the missing `templateJobs`
file.

Key code layout (one directory, 31 files, ~12.6k LOC):

| File | LOC | Role |
|---|---|---|
| `bridge/bridgeMain.ts` | 2999 | Standalone `remote-control` command, `runBridgeLoop`, `runBridgeHeadless` |
| `bridge/replBridge.ts` | 2406 | `initBridgeCore` (bootstrap-free core for REPL + daemon) |
| `bridge/remoteBridgeCore.ts` | 1008 | Env-less bridge core (`initEnvLessBridgeCore`) |
| `bridge/initReplBridge.ts` | 569 | REPL wrapper (gates, auth, title, delegates to core) |
| `bridge/bridgeApi.ts` | 539 | Axios-based client for Environments API |
| `bridge/bridgeUI.ts` | 530 | TUI status display for `remote-control` |
| `bridge/sessionRunner.ts` | 550 | Child-process spawner for session workers |
| `bridge/bridgeMessaging.ts` | 461 | Ingress WS / SSE message parsing & dispatch |
| `bridge/createSession.ts` | 384 | Session create / archive / rename REST calls |
| `bridge/replBridgeTransport.ts` | 370 | v1 WebSocket + v2 SSE+CCR transports |
| `bridge/types.ts` | 262 | Type definitions for configs, handles, errors |

---

## 1. Two front doors

```
┌────────────────────────────────────────────────────────────────┐
│ (A) Standalone:   claude remote-control [--spawn=…]            │
│     entrypoints/cli.tsx:112-162  → bridge/bridgeMain.ts         │
│                                                                │
│ (B) REPL-hosted:  `/remote-control` slash cmd (or auto-start)  │
│     useReplBridge → bridge/initReplBridge.ts                   │
│                   → bridge/replBridge.ts (initBridgeCore)       │
│                     or bridge/remoteBridgeCore.ts (env-less)    │
│                                                                │
│ (C) Daemon worker: claude --daemon-worker remoteControl        │
│     entrypoints/cli.tsx:100-106 → daemon/workerRegistry.js     │
│                                  → bridge/bridgeMain.runBridgeHeadless│
└────────────────────────────────────────────────────────────────┘
```

All three eventually converge on `runBridgeLoop` (for env-based) or
the v2 SSE+CCR transport (for env-less). The split between them is
about which code path builds the config — not about what the poll
loop does.

---

## 2. CLI fast-path dispatch (`entrypoints/cli.tsx`)

`main()` in `cli.tsx:76` dispatches before loading the full CLI to keep
fast-path boot time under the REPL's startup cost. The bridge cases
are:

- **Line 100-106** — `--daemon-worker <kind>` (spawned by the
  supervisor, not user-facing). Feature-gated on `DAEMON`. Imports
  `daemon/workerRegistry.js` → `runDaemonWorker(kind)` which resolves
  `kind==='remoteControl'` to `runBridgeHeadless`.
- **Line 112-162** — `remote-control | rc | remote | sync | bridge`.
  Feature-gated on `BRIDGE_MODE`. Sequence:
  1. `enableConfigs()` — bridge bypasses `init.ts`, so `config.js`
     must be armed manually before any `getGlobalConfig()` call
     (`cli.tsx:116-117`).
  2. OAuth check: `getClaudeAIOAuthTokens()?.accessToken` — fails
     before the GrowthBook gate check because GB has no user context
     without auth and would return a stale/default `false`
     (`cli.tsx:132-141`).
  3. `getBridgeDisabledReason()` → `checkBridgeMinVersion()` — fresh
     GrowthBook values, awaits init
     (`bridge/bridgeEnabled.ts`).
  4. `waitForPolicyLimitsToLoad()` then
     `isPolicyAllowed('allow_remote_control')` — org admins can disable
     remote control via MDM / enterprise policy.
  5. `bridgeMain(args.slice(1))`.

Note no `initSinks()` here — `bridgeMain` calls it internally after
parseArgs so the "multi-session denied" analytics event can be
enqueued before `process.exit(1)` (`bridgeMain.ts:2046-2048`).

---

## 3. Standalone `bridgeMain` flow (`bridge/bridgeMain.ts:1980`)

### Phase 1 — Arg parsing (`parseArgs` at `bridgeMain.ts:1737`)

Arguments supported:
- `--verbose / -v`, `--sandbox / --no-sandbox`
- `--debug-file <path>`, `--session-timeout <seconds>`,
  `--permission-mode <mode>`, `--name <name>`
- `--spawn session|same-dir|worktree` (multi-session mode)
- `--capacity <N>` (max concurrent sessions)
- `--[no-]create-session-in-dir` (pre-create a session at start)
- KAIROS-gated: `--session-id <id>`, `-c / --continue`

Validation rules (all local to `parseArgs`):
- `--capacity` incompatible with `--spawn=session`
- `--session-id` / `--continue` incompatible with spawn flags and
  mutually exclusive
- Each of `--spawn`, `--capacity` may appear only once

### Phase 2 — Pre-loop setup (`bridgeMain.ts:1980-2435`)

1. **`enableConfigs`** + **`initSinks`** — same reason: bridge bypasses
   `init.ts`.
2. **Multi-session gate check** — `isMultiSessionSpawnEnabled()`
   (`bridgeMain.ts:96`) wraps `tengu_ccr_bridge_multi_session`. If
   user passed `--spawn/--capacity/--create-session-in-dir` but gate
   is off → log `tengu_bridge_multi_session_denied`, flush analytics
   (500ms cap), `exit(1)` (`bridgeMain.ts:2055-2076`).
3. **Bootstrap CWD + trust check** — `setCwd`/`setOriginalCwd` then
   `checkHasTrustDialogAccepted()`. Unlike `setup()`, the bridge
   cannot render the interactive trust dialog, so trust must have been
   granted by a prior `claude` run.
4. **OAuth token fetch** — `getBridgeAccessToken()` from
   `bridgeConfig.ts`. Prefers `CLAUDE_BRIDGE_OAUTH_TOKEN` env-var
   override (ant-only), otherwise keychain.
5. **First-time remote dialog** — `readline` prompt that saves
   `remoteDialogSeen` to global config. Skipped on 2nd launch.
6. **`--continue` resolution** (KAIROS-only) —
   `readBridgePointerAcrossWorktreesor()` searches the current
   directory and git worktree siblings for
   `bridge-pointer.json` and chains the resolved session into the
   `--session-id` flow. Pointer directory is tracked in
   `resumePointerDir` so that fatal failures clear the right file
   only (`bridgeMain.ts:2149-2175`).
7. **HTTPS-or-localhost-only guard** — `baseUrl.startsWith('http://')`
   with non-localhost → `exit(1)`. Credential protection.
8. **First-run spawn-mode choice dialog** — if the multi-session gate
   is on AND the project has no saved pref AND worktree is available
   AND stdin is a TTY, a `readline` dialog asks `[1] same-dir` vs
   `[2] worktree`. Saves to `ProjectConfig.remoteControlSpawnMode`.
9. **Effective spawn-mode precedence**:
   `resume > --spawn flag > saved pref > gate default`
   (`bridgeMain.ts:2288-2302`).
10. **`--session-id` resume pre-flight** — proactive OAuth refresh
    (`getBridgeSession` uses raw axios, no 401 retry middleware), then
    `getBridgeSession(resumeSessionId)` to fetch `environment_id`.
    Missing session / missing env → exit after maybe clearing the
    pointer.
11. **Bridge-environment registration** —
    `api.registerBridgeEnvironment(config)` returns
    `environment_id` + `environment_secret`. Fatal 404 → friendly
    "not available for your account" message.
12. **Stale-worker re-queue** — if resume succeeded on the same
    environment, `reconnectSession` is called with both `session_*`
    and `cse_*` ID variants (ccr_v2_compat_enabled makes the lookup
    key variable) (`bridgeMain.ts:2491-2545`).

### Phase 3 — `runBridgeLoop` (`bridgeMain.ts:141-1580`)

This is the 1,400-line poll loop. Core state:

| Map/Set | Key | Value |
|---|---|---|
| `activeSessions` | sessionId | `SessionHandle` |
| `sessionStartTimes` | sessionId | epoch ms |
| `sessionWorkIds` | sessionId | workId |
| `sessionCompatIds` | sessionId (infra/cse_*) | compat/session_* |
| `sessionIngressTokens` | sessionId | session-ingress JWT |
| `sessionTimers` | sessionId | `setTimeout` handle |
| `completedWorkIds` | workId | — (dedup) |
| `sessionWorktrees` | sessionId | `{worktreePath, gitRoot, …}` |
| `timedOutSessions` | sessionId | — (watchdog distinguisher) |
| `titledSessions` | compatId | — (prevent title clobber) |
| `v2Sessions` | sessionId | — (CCR v2 routing flag) |

Loop body sequence per iteration:

1. **`pollForWork`** — long-poll with `loopSignal` (bridge/bridgeApi.ts).
2. **Empty-work path** — choice between three sleeps, picked by
   capacity + GrowthBook:
   - At capacity + `non_exclusive_heartbeat_interval_ms > 0` → nested
     heartbeat loop that sends `heartbeatWork` for each active session
     until capacity changes or `poll_due` deadline hits
     (`bridgeMain.ts:650-730`).
   - At capacity + heartbeat off → slow poll
     (`multisession_poll_interval_ms_at_capacity`).
   - Below capacity → poll at
     `partial_capacity` or `not_at_capacity` interval depending on
     `activeSessions.size > 0`.
3. **Work received** path checks:
   - `completedWorkIds.has(work.id)` → skip (server re-delivered
     stale work before stopWork propagated).
   - `decodeWorkSecret(work.secret)` → if it throws, `stopWork` the
     poisoned item to prevent XAUTOCLAIM re-delivery.
4. **Switch on `work.data.type`**:
   - `'healthcheck'` → ack + log.
   - `'session'`:
     a. Validate `session_id` format.
     b. If already in `activeSessions`, deliver the fresh
        `session_ingress_token` to the child (handles WS drop + server
        re-dispatch). Re-schedule token refresh.
     c. At capacity → break (throttle sleep at bottom of iteration).
     d. **Spawn**:
        - Build `sdkUrl`. CCR v2 path:
          `buildCCRv2SdkUrl` + `registerWorker` with 2× retry;
          `workerEpoch` returned.
          v1 path: `buildSdkUrl(sessionIngressUrl, sessionId)`.
        - `worktree` mode → `createAgentWorktree(bridge-<safeId>)` for
          all sessions *except* the pre-created initial one (stays in
          `config.dir`).
        - `safeSpawn(spawner, opts, dir)` — launches a child
          `claude` process via `sessionRunner.ts`.
        - Start status update ticker.
        - Fetch session title from server concurrently
          (`fetchSessionTitle`); if missing, the `onFirstUserMessage`
          callback derives one from the first user prompt.
        - Start per-session timeout watchdog if
          `sessionTimeoutMs > 0`.
        - Schedule proactive token refresh (v2: `reconnectSession`
          server re-dispatch; v1: deliver fresh OAuth to child).
        - `handle.done.then(onSessionDone(…))` — cleanup hook.
   - default → ack + ignore (forward-compat).
5. **Error handling**:
   - `BridgeFatalError` (401/403/env-expired) → `fatalExit=true`,
     emit telemetry, break loop.
   - Connection errors / 5xx → split backoff tracks
     (`connBackoff` vs `generalBackoff`), 10 min give-up timer.
     Sleep-detection: a gap exceeding 2× the backoff cap resets both
     trackers (laptop wake scenario).
   - All backoff sleeps call `heartbeatActiveWorkItems` first so the
     300s work-lease TTL doesn't expire while the poll is down.

### Phase 4 — Graceful shutdown (`bridgeMain.ts:1403-1579`)

Triggered by `controller.abort()` (SIGINT/SIGTERM):

1. Stop status-update ticker, clear render.
2. Emit `tengu_bridge_shutdown` telemetry.
3. For each active session:
   - `handle.kill()` (SIGTERM).
   - Race against `shutdownGraceMs` (30s default).
   - `handle.forceKill()` (SIGKILL) any stragglers.
4. Clear per-session timeout + token refresh timers.
5. Cleanup any remaining worktrees via `removeAgentWorktree`.
6. `stopWork(…, should_restart=true)` on each workId so the server
   knows the work is done and can re-dispatch if needed.
7. Await `pendingCleanups` to drain in-flight `stopWorkWithRetry`
   and worktree removal promises.
8. **Resumable-shutdown short-circuit** (KAIROS only): if
   `spawnMode === 'single-session'` and we have an
   `initialSessionId` and `!fatalExit`, print
   `"Resume this session by running \`claude remote-control --continue\`"`
   and **return without archive/deregister**. The pointer stays on
   disk.
9. Otherwise: `archiveSession` every session in parallel, then
   `deregisterEnvironment` once.
10. `clearBridgePointer(config.dir)`.

### Exit

`bridgeMain` calls `process.exit(0)` explicitly at `bridgeMain.ts:2767`
because the bridge bypassed `init.ts` (and its graceful shutdown
handler). The `--session-id` resume-fallback path handles `exit(1)`
cases individually.

---

## 4. Headless daemon entry (`runBridgeHeadless` at `bridgeMain.ts:2810`)

Used by the `remoteControl` daemon worker. Strips:
- Readline dialogs (first-run, first-time-remote, spawn-mode pick).
- Stdin key handlers (no `w`/space keys).
- TUI (`createHeadlessBridgeLogger` at `bridgeMain.ts:2968` stubs
  every UI method into a single line-logger).
- `process.exit()` calls.

Adds:
- `BridgeHeadlessPermanentError` class — daemon worker catches this
  and returns `EXIT_CODE_PERMANENT` so the supervisor parks the
  worker instead of backoff-retrying.
- IPC-supplied auth: `opts.getAccessToken` / `opts.onAuth401` come
  from the supervisor's `AuthManager`. The bridge never touches the
  keychain directly in this mode.
- Abort signal: the worker registry passes its `signal` through so a
  supervisor shutdown unwinds the poll loop cleanly.

The shape is a linear subset of `bridgeMain`: chdir → configs →
sinks → trust check → token check → HTTPS check → ingress-URL →
worktree-availability → `registerBridgeEnvironment` → optional
`createBridgeSession` (pre-seed) → `runBridgeLoop`.

---

## 5. REPL-hosted bridge (`initReplBridge.ts` → `replBridge.ts`)

### Entry surface

`initReplBridge(options)` is called via dynamic import from:
- `hooks/useReplBridge.ts` (auto-start when user hits
  `/remote-control`, or on session rehydrate).
- `print.ts` (SDK `-p` mode via `query.enableRemoteControl`).

### Phased gates (`initReplBridge.ts:110-241`)

1. `setCseShimGate(isCseShimEnabled)` — wires `toCompatSessionId` to
   the live GrowthBook gate so the session-ID shim can be turned off
   per-user.
2. `isBridgeEnabledBlocking()` — blocking GrowthBook gate.
3. OAuth present? → else `onStateChange('failed', '/login')` + skip.
4. `waitForPolicyLimitsToLoad()` + `isPolicyAllowed` — same policy
   check as standalone.
5. **Cross-process dead-token backoff** (`initReplBridge.ts:178-187`):
   the global config stores `bridgeOauthDeadExpiresAt` +
   `bridgeOauthDeadFailCount`. Once ≥ 3 consecutive processes have
   failed OAuth refresh against the same `expiresAt`, further launches
   skip silently. Self-clears on `/login` (new token → new
   `expiresAt` → key mismatches).
6. `checkAndRefreshOAuthTokenIfNeeded()` + the post-refresh expiry
   check (`initReplBridge.ts:201-240`) — proactive refresh mirrors
   `bridgeMain.ts:2096` but with extra cross-process dead-token
   persistence.

### Title derivation (`initReplBridge.ts:258-378`)

Precedence for initial title:
`initialName` → `/rename` (session storage) → last meaningful user
message (deriveTitle) → `remote-control-<wordSlug>`.

`onUserMessage(text, bridgeSessionId)` is called by both v1 and v2 cores
on every title-worthy user message:
- Count 1 (if `!hasTitle`): derive placeholder immediately, fire Haiku
  `generateSessionTitle()` for a better replacement.
- Count 3: re-generate over full conversation.
- Returns `true` once count ≥ 3 to signal "done — stop calling me."

Race guards:
- `genSeq` counter prevents an older in-flight Haiku result from
  clobbering a newer one.
- `lastBridgeSessionId` detects env-lost re-creates (v1 only) and
  resets counts.
- `getCurrentSessionTitle(getSessionId())` re-check on every fire
  honors `/rename` between messages.

### Env-less (v2) vs env-based (v1) branch (`initReplBridge.ts:410-452`)

If `isEnvLessBridgeEnabled()` AND `!perpetual`:
- Dynamic-import `remoteBridgeCore.js` → `initEnvLessBridgeCore(…)`.
- Skip the Environments API entirely.

Else (v1 env-based):
- `checkBridgeMinVersion` → gather git context (`getBranch`,
  `getRemoteUrl`) → dispatch to `initBridgeCore` (in `replBridge.ts`).
- KAIROS-guarded: if `isAssistantMode()`, set
  `workerType = 'claude_code_assistant'` so the web UI can filter
  assistant-mode sessions into a separate picker.

### `initBridgeCore` in `replBridge.ts:260`

A bootstrap-free core that takes every bootstrap-read value as a
parameter (cwd, git, auth, title, callbacks, poll-config getter).
Split this way so a daemon caller that never ran `main.tsx` can reuse
the same core with its own auth manager + static poll config.

Key internal flows:
1. **Perpetual-mode prior** — reads `bridge-pointer.json` (source:
   `repl`). If found, reuses `environmentId` + preferred `session_id`
   (`replBridge.ts:305-315`).
2. **`registerBridgeEnvironment`** — always, with
   `reuseEnvironmentId` from prior if perpetual.
3. **`tryReconnectInPlace`** (`replBridge.ts:381`) — if the returned
   env matches the requested one, force-stop stale workers + re-queue
   the prior session. Tries both `session_*` and `cse_*` ID forms.
4. **Fresh `createSession`** if reconnect failed. `previouslyFlushedUUIDs.clear()`
   so the replay stream sends full history again (`replBridge.ts:768`).
5. **Transport build** — v1 WebSocket (`createV1ReplTransport`) unless
   `CLAUDE_CODE_USE_CCR_V2` (ant) or server-side flag flips to v2
   (`createV2ReplTransport`).
6. **Poll loop** — the same shape as `runBridgeLoop`, but single-
   session and local (handle is for *this* process, not a child).
7. **Teardown** returns a `BridgeCoreHandle` superset of
   `ReplBridgeHandle`, adding `getSSESequenceNum()` so daemon callers
   can persist the seq-num across process restarts.

### Differences between v1 core and v2 env-less core

| Aspect | v1 (`replBridge.ts`) | v2 (`remoteBridgeCore.ts`) |
|---|---|---|
| Environments API | Yes (register/poll/ack/heartbeat/deregister) | No |
| Session create | POST /v1/sessions with env_id | POST /v1/code/sessions |
| Worker JWT | From poll work-secret | POST /bridge returns `worker_jwt` + `worker_epoch` |
| Transport | WS (v1) or SSE+CCR (v2 transport on env-based) | Always SSE+CCR |
| Refresh | Poll returns new secret | Re-call /bridge (bumps epoch) + rebuild transport |
| REPL-only? | No (daemon uses this path) | Yes |
| Gate | Default | `tengu_bridge_repl_v2` |
| Perpetual | Supported | Not yet — falls back to v1 when `perpetual:true` |
| Lines | 2406 | 1008 |

Both share `bridgeMessaging.ts` for ingress message handling
(`handleIngressMessage` / `handleServerControlRequest`), `FlushGate`
for ordering initial history vs. live writes, and
`createTokenRefreshScheduler` for JWT renewal.

---

## 6. Support modules inventory

| File | Responsibility |
|---|---|
| `bridgeApi.ts` | Axios client, `BridgeFatalError`, `isExpiredErrorType`, `isSuppressible403`, `validateBridgeId`. Thin wrapper for register/poll/ack/stopWork/heartbeat/reconnect/archive/deregister endpoints. |
| `bridgeConfig.ts` | `getBridgeAccessToken`, `getBridgeBaseUrl`, `getBridgeTokenOverride` — env-var hooks for ant local dev. |
| `bridgeDebug.ts` | `wrapApiForFaultInjection`, `registerBridgeDebugHandle`, `injectBridgeFault` — ant-only (`/bridge-kick`) for failure simulation. |
| `bridgeEnabled.ts` | GrowthBook gate checks (`isBridgeEnabledBlocking`, `getBridgeDisabledReason`, `isCseShimEnabled`, `isEnvLessBridgeEnabled`, `checkBridgeMinVersion`). |
| `bridgeMessaging.ts` | Parses ingress stream frames; routes to `onInboundMessage`, permission responses, server control requests (interrupt, set-model, set-permission-mode). Maintains UUID dedup. |
| `bridgePermissionCallbacks.ts` | Adapter between bridge permission prompts and the REPL permission prompt system. |
| `bridgePointer.ts` | Read/write/clear `bridge-pointer.json` for crash-recovery and `--continue`. `readBridgePointerAcrossWorktrees` fans out to git worktree siblings. |
| `bridgeStatusUtil.ts` | `formatDuration`, `formatDelay` helpers. |
| `bridgeUI.ts` | Interactive `remote-control` TUI: status line, QR-code toggle, session list, reconnect countdown. |
| `capacityWake.ts` | Signal that wakes at-capacity sleeps when a session ends. |
| `codeSessionApi.ts` | POST /v1/code/sessions/{id}/bridge endpoint wrapper for env-less path. |
| `createSession.ts` | POST /v1/sessions, GET /v1/sessions/{id}, archive, updateTitle — with ccr-byoc headers + org UUID. |
| `debugUtils.ts` | `describeAxiosError`, `logBridgeSkip` telemetry helpers. |
| `envLessBridgeConfig.ts` | GrowthBook-backed config for env-less path (timeouts, retry caps, heartbeat interval, JWT refresh buffer). |
| `flushGate.ts` | Queue gate: buffers live writes until initial-history flush completes so the server sees `[history…, live…]` in order. |
| `inboundAttachments.ts` / `inboundMessages.ts` | Normalize inbound user messages (attachments, formatting) for REPL injection. |
| `jwtUtils.ts` | `createTokenRefreshScheduler` — schedules per-session callbacks 5min before JWT expiry. Works for both v1 and v2. |
| `pollConfig.ts` / `pollConfigDefaults.ts` | Poll-interval config: `not_at_capacity`, `partial_capacity`, `at_capacity`, `non_exclusive_heartbeat_interval_ms`, `reclaim_older_than_ms`. GrowthBook + in-process TTL cache. |
| `replBridgeHandle.ts` | Type export for `ReplBridgeHandle` — used by hooks that can't import `replBridge.ts` without transitive cost. |
| `replBridgeTransport.ts` | `createV1ReplTransport` (WebSocket to session-ingress) and `createV2ReplTransport` (SSE read + CCRClient write). |
| `sessionIdCompat.ts` | `toCompatSessionId` / `toInfraSessionId` — UUID-preserving retag between `session_*` (compat) and `cse_*` (infra) forms. GrowthBook-gated via `setCseShimGate`. |
| `sessionRunner.ts` | `createSessionSpawner` — spawns child `claude` processes with `--sdk-url`, manages stdin/stdout, tracks activity. |
| `trustedDevice.ts` | `getTrustedDeviceToken` — per-device identity token sent on every bridge call. |
| `types.ts` | `BridgeConfig`, `BridgeLogger`, `SessionHandle`, `SpawnMode`, `BridgeWorkerType`, constants (`DEFAULT_SESSION_TIMEOUT_MS`, `BRIDGE_LOGIN_ERROR`). |
| `workSecret.ts` | `decodeWorkSecret` (extract session-ingress JWT from encrypted blob), `buildSdkUrl`, `buildCCRv2SdkUrl`, `registerWorker`. |

---

## 7. Integration points

- **Auth (`utils/auth.ts`)** — `getClaudeAIOAuthTokens`,
  `checkAndRefreshOAuthTokenIfNeeded`, `handleOAuth401Error`,
  `clearOAuthTokenCache`. Both standalone and REPL paths must
  proactively refresh before the first API call. REPL adds a dead-
  token cross-process backoff (global config).
- **Config (`utils/config.ts`)** — `enableConfigs()` is mandatory in
  bridge fast-paths. Project-level `remoteControlSpawnMode` persists
  the user's choice. Global `remoteDialogSeen`, `bridgeOauthDeadExpiresAt`,
  `bridgeOauthDeadFailCount` drive dialogs + backoff.
- **Policy (`services/policyLimits`)** — `allow_remote_control` MDM
  policy. Blocking load in standalone; policy may be stale-cached at
  the REPL path's mount time.
- **Worktree (`utils/worktree.ts`)** — `createAgentWorktree` +
  `removeAgentWorktree` for spawn-per-worktree mode. `hasWorktreeCreateHook`
  + `findGitRoot` for availability check.
- **Session-ingress (`utils/sessionIngressAuth.ts`)** —
  `updateSessionIngressAuthToken` propagates refreshed JWTs to the
  ingress WS reader.
- **Concurrent sessions (`utils/concurrentSessions.ts`)** —
  `updateSessionBridgeId` records the bridge session ID in
  `~/.claude/sessions/` so `ps`/`attach` subcommands can find it.
- **Analytics (`services/analytics`)** — dozens of `tengu_bridge_*`
  events; `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`
  is an ESLint-enforced opaque tag that confirms the value is safe
  to log without PII review.
- **Cleanup registry (`utils/cleanupRegistry.ts`)** — both cores
  register teardown functions so `gracefulShutdown` unwinds them.
- **Commands registry (avoidance!)** — `replBridge.ts` and
  `remoteBridgeCore.ts` are engineered to NOT import `src/commands.ts`.
  `BridgeCoreParams` injects `toSDKMessages`, `createSession`,
  `archiveSession`, `onAuth401`, `getPollIntervalConfig` as callbacks
  to avoid dragging the 1300-module command + React tree into the
  Agent SDK bundle. `initReplBridge.ts` is the wrapper that pulls
  those in and forwards them — the REPL can afford the weight; the
  daemon cannot.

---

## 8. State diagram (from `BridgeState` type, `replBridge.ts:83`)

```
           initReplBridge entry
                  ↓
     [gates, auth, title, git]
                  ↓
            initBridgeCore
                  ↓
        registerBridgeEnvironment
                  ↓
  ┌──────────────┬───────────────┐
  │              │               │
  failed      ready        (error path)
  ↓              ↓
(returns null)  createBridgeSession
                ↓
           startTransport + flushHistory
                ↓
           (flush finishes, cb fires)
                ↓
            connected
                ↓
         (server interrupt / net blip)
                ↓
           reconnecting  ──►  (success) connected
                ↓
           (give-up)
                ↓
             failed
```

`BridgeState` transitions are surfaced via `onStateChange`; the REPL
uses them to drive the `/remote-control` status UI (green dot vs.
amber reconnect spinner vs. red `/login` prompt).

---

## 9. Multi-session lifecycle — what happens when

Standalone `claude remote-control --spawn=worktree --capacity=4`:

1. Poll loop idles, `activeSessions` empty.
2. Web UI creates a new session → server dispatches `work` with
   `type:'session'`, `session_id: cse_abc…`.
3. Poll returns this work. `ackWork()`, decode secret, build
   `sdkUrl` via `buildCCRv2SdkUrl`, `registerWorker` (v2 path),
   `createAgentWorktree('bridge-abc')` at `/tmp/worktree-abc`.
4. `safeSpawn` launches `claude --sdk-url=<url> --sandbox …`. Child
   boots, connects to session-ingress, reports activity via stdout
   IPC.
5. Parent tracks `handle`, starts timeout watchdog, schedules token
   refresh, fetches server title.
6. Child's first tool_use event flows inbound via the WS → parent's
   logger updates the TUI.
7. User sends "exit" → child completes → `handle.done` resolves →
   `onSessionDone('completed')` → `stopWork(workId)` + `archiveSession(compatId)` +
   `removeAgentWorktree`. Loop stays up.
8. Server dispatches next session → repeat.
9. At `activeSessions.size === 4`, the loop enters heartbeat mode.
10. User hits Ctrl+C → `controller.abort()` → all children SIGTERM,
    30s grace, SIGKILL, stopWork × 4, archive × 4,
    deregisterEnvironment, clearBridgePointer, `process.exit(0)`.

---

## 10. Templates subsystem — **MISSING FROM THIS SNAPSHOT**

`cli.tsx:211-222` dispatches `claude new|list|reply` (gated by
`feature('TEMPLATES')`) to:

```ts
const { templatesMain } = await import('../cli/handlers/templateJobs.js')
await templatesMain(args)
process.exit(0)  // mountFleetView's Ink TUI can leak event-loop handles
```

**However, `src/cli/handlers/templateJobs.js` does not exist in this
source snapshot.** Verified by:
- Directory listing of `src/cli/handlers/` shows only `agents.ts`,
  `auth.ts`, `autoMode.ts`, `mcp.tsx`, `plugins.ts`, `util.tsx`.
- `find src -name 'templateJobs*'` returns nothing.
- `grep templatesMain` returns only the one call-site above.
- `grep FleetView / mountFleetView` returns only the cli.tsx comment
  and an unrelated `useCopyOnSelect` hook.

From the call site and commit message we know:
- Subcommands: `new`, `list`, `reply`.
- Uses Ink TUI (comment: "mountFleetView's Ink TUI can leave event
  loop handles that prevent natural exit").
- Gated by `feature('TEMPLATES')` — meaning tree-shaken out of
  external builds.

Likely shape (inferred, NOT verified from source):
- `new` — create a new template job from a template file.
- `list` — show running / recent template jobs as a fleet view.
- `reply` — inject a reply into a running template job.

**Recommendation.** Either a later wave of this analysis will see the
file added to `src/cli/handlers/`, OR the templates subsystem has been
stripped from the public release and only exists in an internal
Anthropic build. Ask the bundling team whether `TEMPLATES` is
configured to off for this snapshot's source dump.

---

## 11. Notable invariants and subtleties

- **Init-bypass pattern is deliberate.** The bridge fast-path skips
  `init.ts`'s mTLS/proxy/shutdown/OAuth plumbing to minimize boot
  latency. Every "why is this here and not in init?" has the same
  answer: init runs for *every* subcommand; the bridge needs only a
  narrow subset, and duplicates the narrow subset (`enableConfigs`,
  `initSinks`, `setOriginalCwd`, `checkHasTrustDialogAccepted`) to
  avoid paying the cost for `doctor`, `mcp`, `update`, etc.
- **`process.exit(0)` in `bridgeMain`** (line 2767) is required —
  nothing else cleans up the REPL's bootstrap side-effects.
- **`session_*` vs `cse_*` shim** — anywhere a session id crosses an
  API boundary, it may need retagging depending on whether the server
  has `ccr_v2_compat_enabled` flipped on. `toCompatSessionId` /
  `toInfraSessionId` are UUID-preserving — same underlying 128 bits,
  different prefix. Writers MUST call these at every boundary; the
  bridge has ~30 call sites.
- **Env-less is REPL-only.** `--bare`, `print`, daemon, and the
  standalone `remote-control` command all stay on env-based. Only
  interactive `/remote-control` inside a REPL takes the env-less
  path.
- **Title updates are cosmetic** — only claude.ai uses them. They
  never reach the model. This matters because Haiku-based
  `generateSessionTitle` can take 1-15 s; if it times out, the
  placeholder sticks and nothing else breaks.
- **Heartbeat composes with poll.** The "at-capacity heartbeat loop"
  doesn't replace polling — it breaks out to poll at `poll_due`
  deadlines. Setting `atCapMs=0` with heartbeat on means
  "heartbeat-only indefinitely until a session ends or abort".
- **Sleep detection uses `2× connCapMs` threshold** — laptop
  wake/sleep with 60s max backoff would be indistinguishable from a
  20-minute outage without this.
- **`fatalExit` flag** — skips the KAIROS resumable-shutdown message
  because `/remote-control --continue` couldn't possibly work after
  a 10-minute give-up or env-expired error.
- **`pendingCleanups` tracker** — `stopWork` retries and worktree
  removal are fire-and-forget from the happy path, but a shutdown
  caller must await them before `process.exit()` or they're killed
  mid-flight. The `trackCleanup(p)` helper is the discipline.
- **Ordering: `!work` branch + capacity throttle** — stale
  redeliveries and decode failures both hit the "work != null → skip
  the `!work` sleep" fast path. Without the explicit sleep inside
  their `continue` branches, the bridge would tight-loop at HTTP
  speed against a persistent poison pill.

---

## 12. Follow-up deep-dive recommendations

1. **Session-ingress protocol deep dive.** `replBridgeTransport.ts`
   (370 LOC) plus `bridgeMessaging.ts` (461 LOC) define the v1 WS
   and v2 SSE+CCR wire format. Worth a dedicated write-up on message
   types, sequence-numbering, reconnection with `from_sequence_num`,
   and CCRClient's `reportState`.
2. **Token refresh scheduler internals.** `jwtUtils.ts` (256 LOC)
   uses a per-session schedule with refresh-buffer windows and
   interacts with three different "refresh" flows (direct OAuth
   delivery, server re-dispatch via `reconnectSession`, `/bridge`
   epoch bump). Subtle race conditions when laptop wake overlaps
   proactive refresh + 401.
3. **`sessionRunner.ts` — child process shape.** How
   `--sdk-url`/`--sandbox`/`--permission-mode` child args are
   wired, stdout parsing for activity events, how
   `onFirstUserMessage` closure reaches into the child, kill-tree
   semantics.
4. **`FlushGate` + initial-history ordering.** The subtlety of
   "server must see history BEFORE live writes" plus
   `recentPostedUUIDs` dedup for server-echoed history is load-
   bearing for correctness.
5. **Daemon-worker path (cross-ref with task #8).** The supervisor
   side of `runBridgeHeadless`: how `daemon/workerRegistry.js`
   dispatches `kind==='remoteControl'`, IPC auth injection, exit-
   code handling for `BridgeHeadlessPermanentError`.
6. **Templates system — ONCE THE FILE APPEARS.** `templateJobs.js`
   + `mountFleetView` + `TEMPLATES` gate. Currently analyzable only
   from the one-line dispatch at `cli.tsx:211-222`.

---

## 13. Code references quick index

| Concern | File:line |
|---|---|
| Bridge fast-path dispatch | `src/entrypoints/cli.tsx:108-162` |
| Daemon-worker fast-path | `src/entrypoints/cli.tsx:100-106` |
| Templates fast-path (stub) | `src/entrypoints/cli.tsx:211-222` |
| `bridgeMain` entry | `src/bridge/bridgeMain.ts:1980` |
| `parseArgs` | `src/bridge/bridgeMain.ts:1737` |
| `runBridgeLoop` | `src/bridge/bridgeMain.ts:141` |
| At-capacity heartbeat | `src/bridge/bridgeMain.ts:650-730` |
| Session spawn | `src/bridge/bridgeMain.ts:859-1204` |
| Graceful shutdown | `src/bridge/bridgeMain.ts:1403-1579` |
| `runBridgeHeadless` | `src/bridge/bridgeMain.ts:2810` |
| `createHeadlessBridgeLogger` | `src/bridge/bridgeMain.ts:2968` |
| `initReplBridge` entry | `src/bridge/initReplBridge.ts:110` |
| Dead-token cross-process backoff | `src/bridge/initReplBridge.ts:178-240` |
| Title derivation | `src/bridge/initReplBridge.ts:258-378` |
| v1/v2 branch | `src/bridge/initReplBridge.ts:410-452` |
| `initBridgeCore` | `src/bridge/replBridge.ts:260` |
| `tryReconnectInPlace` | `src/bridge/replBridge.ts:381` |
| `initEnvLessBridgeCore` | `src/bridge/remoteBridgeCore.ts:140` |
| `BridgeCoreParams` type | `src/bridge/replBridge.ts:91-221` |
| `BridgeState` type | `src/bridge/replBridge.ts:83` |
| Types / constants | `src/bridge/types.ts` |
| Missing file (flagged) | `src/cli/handlers/templateJobs.js` (does not exist in snapshot) |
