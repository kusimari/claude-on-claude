# Daemon Architecture and Operation

**Task**: #8 — "Analyze daemon architecture and operation"
**Owner**: analyst-1
**Wave / Phase**: 1 / 4
**Status**: Completed with a significant caveat (see §1)

---

## 1. Primary Finding — Daemon source is excluded from the extracted tree

The entrypoint `src/entrypoints/cli.tsx` has two dynamic imports into a `daemon/`
sibling module:

- `src/entrypoints/cli.tsx:100-106` — `--daemon-worker <kind>` worker
  fast-path: `await import('../daemon/workerRegistry.js')` and
  `await runDaemonWorker(args[1])`.
- `src/entrypoints/cli.tsx:164-180` — `claude daemon [subcommand]`
  supervisor fast-path: `await import('../daemon/main.js')` and
  `await daemonMain(args.slice(1))`.

Both imports are gated by `feature('DAEMON')`. **Neither target file exists
in the extracted source.** `ls src/daemon` returns `No such file or
directory`. A Grep for `daemon` across `src/` shows 28 matching files — all
of them *reference* the daemon without providing its implementation.

This is a strong signal that the daemon module was feature-gated out during
the extraction build (the `feature('DAEMON')` branch's imports were
tree-shaken, leaving only the caller-side dispatch). Nothing in the
extracted source actually runs as the daemon supervisor or its worker
registry.

Consequences for this analysis:

- I can accurately describe the **contract** daemon mode exposes to the
  rest of the CLI (entrypoint dispatch, headless worker bridge entry, auth
  hand-off, session-kind telemetry, cron integration, error taxonomy).
- I **cannot** describe the supervisor's internal loop (spawn/backoff
  scheduling, per-worker lifecycle, crash detection, IPC implementation,
  `daemon.json` schema). Those live behind `daemon/main.ts` and
  `daemon/workerRegistry.ts`, which this tree does not contain.

**Recommended follow-up: re-extract with the `DAEMON` feature gate
enabled, or source the daemon module separately.** Without it, any
analysis of worker scheduling, auth-token sharing, and graceful
supervisor shutdown is inference from call-site comments.

---

## 2. What daemon mode is, from the call sites

Daemon mode is a **long-running supervisor process** (`claude daemon`)
that owns and respawns one or more **worker children**, each launched as
`claude --daemon-worker <kind>`. Workers are lean — they skip
`enableConfigs()` and `initSinks()` at the CLI dispatcher layer
(`cli.tsx:97 — "workers are lean"`) and call those themselves only if
they need them (see the bridge headless worker in §4, which *does* call
both).

Two things make daemon mode distinct from interactive/bridge/bg:

1. **Supervised lifecycle.** The parent is a persistent process the user
   started explicitly (`claude daemon`). Children are not user-facing;
   the user never types `claude --daemon-worker`, the supervisor spawns
   it. The supervisor is responsible for restart-on-crash with backoff
   and for **not** restarting on configuration failures.
2. **Shared auth via IPC.** The supervisor owns an `AuthManager` (per
   `bridgeMain.ts:2804`). Workers get tokens from it over IPC rather than
   re-reading the on-disk OAuth state themselves. This avoids lock
   contention, avoids each worker independently triggering refresh, and
   lets a single refresh benefit all workers.

Worker *kinds* are selected by the string passed as the second CLI arg
to `--daemon-worker`. Only one kind appears in the extracted tree by
name: `remoteControl`, documented in `bridgeMain.ts:2800` as the
non-interactive bridge entrypoint. Other kinds (assistant/Kairos,
possibly `cronWorker` — see §6) are named in comments but their
run() functions live in the missing `workerRegistry.ts`.

---

## 3. Entrypoint dispatch — how daemon paths are taken

`src/entrypoints/cli.tsx`, both paths run *before* any config or sink
initialization for a reason the comments spell out.

### 3.1 Worker fast-path (`cli.tsx:95-106`)

```ts
if (feature('DAEMON') && args[0] === '--daemon-worker') {
  const { runDaemonWorker } = await import('../daemon/workerRegistry.js')
  await runDaemonWorker(args[1])
  return
}
```

Key properties:
- Comes **before** the daemon subcommand check. Comment at
  `cli.tsx:96` — "Must come before the daemon subcommand check: spawned
  per-worker, so perf-sensitive."
- No `enableConfigs()`, no `initSinks()` here. Workers that need them
  opt in inside their own `run()` function. The bridge headless worker
  does — see `bridgeMain.ts:2824-2829`.
- `runDaemonWorker(args[1])` is a dispatcher keyed by worker kind.
  Concrete worker implementations presumably live in
  `daemon/workerRegistry.ts` (missing).
- `feature('DAEMON')` gates the entire branch so external / non-daemon
  builds DCE the import reference.

### 3.2 Supervisor fast-path (`cli.tsx:164-180`)

```ts
if (feature('DAEMON') && args[0] === 'daemon') {
  profileCheckpoint('cli_daemon_path')
  const { enableConfigs } = await import('../utils/config.js')
  enableConfigs()
  const { initSinks } = await import('../utils/sinks.js')
  initSinks()
  const { daemonMain } = await import('../daemon/main.js')
  await daemonMain(args.slice(1))
  return
}
```

Key properties:
- Supervisor **does** run `enableConfigs()` and `initSinks()` — it
  lives a long time and needs proper analytics attachment.
- `profileCheckpoint('cli_daemon_path')` — the supervisor's cold-start
  time is tracked by the profile instrumentation (worker fast-path is
  not, because it's expected to be sub-100ms).
- `args.slice(1)` lets the daemon accept subcommands: likely
  `start`/`stop`/`status`/`install` style; the exact set is in the
  missing `daemon/main.ts`.
- No `--background` detach is visible here. Either `daemonMain` forks
  itself into the background internally, or the user is expected to
  drive backgrounding externally (systemd, launchd, tmux).

### 3.3 Ordering invariant

The dispatcher's current order is:
1. `--config-exporter` (`cli.tsx:~70`)
2. `--chrome-native-host`
3. `--computer-use-mcp`
4. `--daemon-worker` ← worker fast-path
5. `remote-control` / `rc` / `bridge` aliases
6. `daemon` ← supervisor fast-path
7. `ps` / `logs` / `attach` / `kill` / `--bg` / `--background`
8. (default) interactive REPL

Swapping (4) and (6) would still function but comment at `cli.tsx:96-97`
suggests the worker fast-path is deliberately earliest-plausible to
minimize worker cold-start cost.

---

## 4. The bridge-headless worker (the one worker kind we can analyze)

`src/bridge/bridgeMain.ts:2770-2965` implements `runBridgeHeadless`,
the entrypoint for the `remoteControl` worker kind. This is the only
worker whose full implementation is in the extracted tree.

Shape of the boundary:

### 4.1 `BridgeHeadlessPermanentError` (`bridgeMain.ts:2778-2783`)

```ts
export class BridgeHeadlessPermanentError extends Error { … }
```

Comment documents the error-taxonomy contract with the supervisor:

> Thrown by runBridgeHeadless for configuration issues the supervisor
> should NOT retry (trust not accepted, worktree unavailable,
> http-not-https). The daemon worker catches this and exits with
> EXIT_CODE_PERMANENT so the supervisor parks the worker instead of
> respawning it on backoff.

That comment is the single load-bearing description of the supervisor's
error-classification contract in the entire extracted codebase. The
scheme:

- **Permanent** → exit `EXIT_CODE_PERMANENT` → supervisor "parks"
  the worker (does not respawn; exposes a diagnostic).
- **Transient** → non-permanent exit → supervisor respawns with
  backoff.

`EXIT_CODE_PERMANENT` is referenced only in this comment — the concrete
value is inside `daemon/workerRegistry.ts` (missing).

### 4.2 `HeadlessBridgeOpts` (`bridgeMain.ts:2785-2797`)

```ts
type HeadlessBridgeOpts = {
  dir: string
  name?: string
  spawnMode: 'same-dir' | 'worktree'
  capacity: number
  permissionMode?: string
  sandbox: boolean
  sessionTimeoutMs?: number
  createSessionOnStart: boolean
  getAccessToken: () => string | undefined
  onAuth401: (failedToken: string) => Promise<boolean>
  log: (s: string) => void
}
```

This type is the contract the supervisor writes against. Observations:

- `getAccessToken` is a **function**, not a token string. The
  supervisor's `AuthManager` owns the live token; the worker calls back
  for it each time. This is the IPC hand-off mentioned at
  `bridgeMain.ts:2804`.
- `onAuth401` returns `Promise<boolean>` — true means the refresh
  succeeded and the caller may retry; false means the token is
  unrecoverable. Feeds into the "transient token" / "permanent
  bad-credential" classification upstream.
- `log` is a callback (not a logger instance). The supervisor routes
  the worker's log lines somewhere — likely a per-worker log file
  surfaced via `claude daemon status`.
- `capacity`, `spawnMode`, `sessionTimeoutMs` are the tunables
  presumably populated from `daemon.json` by the supervisor.

### 4.3 `runBridgeHeadless` (`bridgeMain.ts:2810-2965`)

Shape of the worker body:

1. **`process.chdir(dir)` + `setOriginalCwd(dir)` + `setCwdState(dir)`**
   — workers inherit the supervisor's CWD, so the worker explicitly
   repositions bootstrap CWD state so git utilities resolve against
   the right repo. (`bridgeMain.ts:2815-2822`)
2. **`enableConfigs()` + `initSinks()`** — the worker-level init the
   entrypoint deliberately skipped. (`bridgeMain.ts:2824-2829`)
3. **Permanent-exit preflight**:
   - `checkHasTrustDialogAccepted` → `BridgeHeadlessPermanentError`
     if workspace is not trusted. (`2831-2835`)
   - `baseUrl` is HTTP and not localhost →
     `BridgeHeadlessPermanentError`. (`2844-2852`)
   - `spawnMode === 'worktree'` and the dir is neither a git repo
     nor has WorktreeCreate hooks → `BridgeHeadlessPermanentError`.
     (`2864-2872`)
4. **Transient preflight**:
   - Missing access token → plain `Error(BRIDGE_LOGIN_ERROR)` (not
     permanent) — "supervisor's AuthManager may pick up a token on
     next cycle". (`2837-2840`)
5. **Config + registration**:
   - Builds `BridgeConfig` with `bridgeId = randomUUID()` and
     `environmentId = randomUUID()`. (`2879-2894`)
   - `api.registerBridgeEnvironment(config)` — failure is classified
     **transient**: "let supervisor backoff-retry". (`2908-2913`)
6. **Session spawner + initial session creation** —
   `createSessionSpawner`, optional `createBridgeSession` with
   `createSessionOnStart`. Pre-creation failure is logged but
   non-fatal. (`2916-2951`)
7. **`runBridgeLoop(...)`** — the main worker loop, also at
   `bridgeMain.ts:~265+`. Workers resolve cleanly when the `signal`
   aborts (`bridgeMain.ts:2808` docstring).

### 4.4 Token refresh inside the worker loop (`bridgeMain.ts:279-313`)

Even inside the worker body, the token hand-off continues to matter.
`createTokenRefreshScheduler` schedules a proactive refresh 5min before
the ingress JWT expires. For v2 sessions it calls `api.reconnectSession`
instead of stamping a new token on the transport:

> (CC-1263: without this, v2 daemon sessions silently die at ~5h since
> the server does not auto-re-dispatch ACK'd work on lease expiry).

`bridgeMain.ts:282` explicitly names "v2 daemon sessions" as the case
this path exists for — a concrete past incident in the daemon worker
path.

### 4.5 Fatal-exit marker (`bridgeMain.ts:331`)

```ts
// Set by BridgeFatalError and give-up paths so the shutdown block can
// skip the resume message (resume is impossible after env expiry/auth
// failure/sustained connection errors).
let fatalExit = false
```

Another touchpoint with the permanent/transient taxonomy — some
runtime failures (not just preflight) also force a non-retryable exit.
The supervisor side presumably maps `fatalExit` to
`EXIT_CODE_PERMANENT` too.

---

## 5. Session-kind telemetry (`concurrentSessions.ts`)

`src/utils/concurrentSessions.ts` is where the daemon's presence
becomes observable to `claude ps`:

- `SessionKind = 'interactive' | 'bg' | 'daemon' | 'daemon-worker'`
  (`concurrentSessions.ts:18`).
- `envSessionKind()` reads `CLAUDE_CODE_SESSION_KIND` from `process.env`
  (`concurrentSessions.ts:31-37`) and is gated by `feature('BG_SESSIONS')`.
  Only `bg`, `daemon`, `daemon-worker` are accepted values;
  `interactive` is the default and is written by `registerSession()`
  without needing an env var.
- Comment at `concurrentSessions.ts:27-30`:

  > Kind override from env. Set by the spawner (`claude --bg`, daemon
  > supervisor) so the child can register without the parent having to
  > write the file for it — cleanup-on-exit wiring then works for free.

  That tells us **the supervisor sets `CLAUDE_CODE_SESSION_KIND=daemon`
  on its own process and `CLAUDE_CODE_SESSION_KIND=daemon-worker` on each
  child it spawns**. Combined with the normal `registerSession()`
  PID-file flow, this makes the supervisor and every worker show up
  individually in `claude ps` with the right kind tag.
- `registerSession()` writes to `~/.claude/sessions/<pid>.json` with
  `kind` populated from `envSessionKind()`.
  (`concurrentSessions.ts:59-109`)
- `countConcurrentSessions()` reads back the registry and prunes
  stale PID files (`concurrentSessions.ts:168-204`). WSL is skipped for
  the prune to avoid falsely sweeping Windows-native Claude sessions
  sharing `~/.claude/sessions/`.

`getAgentId() != null` short-circuits `registerSession()` — teammates
and subagents don't appear in ps, which keeps swarms from polluting
the process list. So daemon workers register, but Agent subagents
inside them do not.

---

## 6. Cron / scheduled-task integration

`src/utils/cronScheduler.ts` has explicit daemon affordances:

- `assistantMode?: boolean` (`cronScheduler.ts:75`) — when true,
  `getScheduledTasksEnabled()` polling is skipped and `enable()` runs
  immediately. Comment:

  > Required for Agent SDK daemon callers.

- `onFireTask` (`cronScheduler.ts:77-78`) — when provided, receives the
  full `CronTask` on normal fires instead of just the prompt string.
  Comment:

  > Lets daemon callers see the task id/cron/etc instead of just the
  > prompt string.

- Permanent-task semantics (`cronScheduler.ts:123-124`):

  > The daemon cron worker uses `t => t.permanent` so non-permanent
  > tasks in the same scheduled_tasks.json are untouched.

This confirms there is at least one additional worker kind beyond
`remoteControl` — the **daemon cron worker** — even though its
concrete `run()` function isn't in the extracted tree. It partitions
the scheduled_tasks.json by the `permanent` flag so other callers
(interactive REPL, `--bg`) can coexist without firing the daemon's
tasks.

`src/utils/cronTasksLock.ts:34-39` separately notes that out-of-REPL
daemon callers must supply their own `lockIdentity` (a random UUID
captured at daemon startup), since they don't have bootstrap
`getSessionId()` state. Another piece of "daemon is not a REPL"
scaffolding.

---

## 7. Kairos / assistant-mode integration (`main.tsx`)

The `claude assistant` install / invoke flow is a **consumer** of the
daemon. `src/main.tsx` shows how interactive sessions interact with an
already-running daemon:

- `src/main.tsx:80` — `assistantModule = feature('KAIROS') ?
  require('./assistant/index.js') as … : null`. The only file in the
  extracted `src/assistant/` tree is `sessionHistory.ts`, so most of
  the assistant module is similarly gated out.
- `src/main.tsx:1052-1056` —

  > `--assistant` (Agent SDK daemon mode): force the latch before
  > isAssistantMode() runs below. The daemon has already checked
  > entitlement — don't make the child re-check tengu_kairos.

  So `--assistant` is a hidden CLI flag (`main.tsx:3842`:
  `'Force assistant mode (Agent SDK daemon use)'`, `hideHelp()`) the
  daemon sets when spawning its assistant worker child, signalling
  "skip Kairos entitlement recheck, parent already did it".

- `src/main.tsx:1075` —

  > (max ~5s). --assistant skips the gate entirely (daemon is
  > pre-entitled).

- `src/main.tsx:3286-3292` — the `claude assistant` install flow
  times out its success message with

  > The daemon needs a few seconds to spin up its worker and
  > establish a bridge session before discovery will find it.

So the **assistant feature's runtime architecture is**: the daemon
runs an assistant worker child which itself is a bridge session; the
user's `claude assistant` invocation discovers that session via the
bridge environment registry and attaches as a *viewer*.

- `src/components/StatusLine.tsx:31-33` —

  > Assistant mode: statusline fields (model, permission mode, cwd)
  > reflect the REPL/daemon process, not what the agent child is
  > actually running. Hide it.

- `src/hooks/useRemoteSession.ts:100-101,128-131` — event-sourced
  task count for the "REMOTE daemon child", viewer's own
  `AppState.tasks` is empty in that mode.

This is an important mental model for the daemon: when `claude assistant`
is running, there are at minimum **three** process layers:

1. `claude daemon` — the supervisor
2. `claude --daemon-worker assistant` (inferred kind name) — the
   assistant child, itself running the Kairos/Agent-SDK team-of-
   agents
3. `claude assistant` — the user's interactive viewer, which is
   *not* a daemon process but a normal REPL in viewer-mode that
   speaks to the worker via the bridge session

---

## 8. Other daemon-awareness scattered through the codebase

The remaining Grep hits round out the picture:

- `src/utils/sinks.ts:5-8` — documents that the daemon bypasses
  `setup()` and calls `initSinks()` directly (consistent with
  `cli.tsx:174`).
- `src/utils/conversationRecovery.ts:2625-2627` — stale-session
  warnings are "particularly loud for daemons that respawn
  frequently". Points at the reality that the daemon does churn
  children.
- `src/utils/conversationRecovery.ts:487-489` — `--continue` skips
  live `--bg`/daemon sessions so the user doesn't accidentally
  hijack a background worker's transcript.
- `src/state/AppStateStore.ts:126-131` — background task count in
  `claude assistant` is event-sourced from the REMOTE daemon child's
  WS stream; the local AppState is always empty.
- `src/bridge/replBridge.ts:~87-88,~113-114,~133-134,~151-152,
  ~172-207,~254-255,~277-278,~760,~795-796,~1034-1036,~1490-1491,
  ~1506-1508,~1587-1589,~1600-1602,~1689,~1760-1762` — bridge-core
  has dozens of daemon-aware branches. The design intent is explicit
  at `replBridge.ts:87-89`: the bridge library exposes a **superset**
  interface (`BridgeCoreParams`) that both the REPL and a daemon
  caller can populate; the REPL fills most slots from bootstrap
  state, the daemon fills them itself from its own state.
- `src/utils/forkedAgent.ts:92` — analytics label `'supervisor'` is
  a first-class source category.
- `src/upstreamproxy/upstreamproxy.ts:13,139` — the upstream proxy is
  designed for "supervisor restart can retry" failure modes, likely
  the same daemon supervisor (or conceptually similar).

---

## 9. How daemon mode differs from interactive (the deliverable)

| Concern | Interactive REPL | Daemon supervisor | Daemon worker (`remoteControl`) |
| --- | --- | --- | --- |
| Entrypoint | `default` branch of `cli.tsx` → `setup()` → REPL | `cli.tsx:164-180` → `daemonMain` | `cli.tsx:100-106` → `runDaemonWorker(kind)` |
| Config init | `setup()` calls `enableConfigs()` | Dispatcher calls it inline | **Worker calls it inside** (`bridgeMain.ts:2827`) |
| Sinks init | `setup()` calls `initSinks()` | Dispatcher calls it inline | **Worker calls it inside** (`bridgeMain.ts:2829`) |
| UI | Ink TUI | None | None (`bridgeMain.ts:2802` "no TUI") |
| Stdin | Raw-mode key handlers | Supervisor's own protocol (not in extracted tree) | None — log callback only |
| Auth | Reads OAuth from `~/.claude/` directly | Owns `AuthManager`; hands out to workers via IPC | Calls `opts.getAccessToken()`, which crosses IPC to supervisor |
| Exit | `process.exit()` via `exit.tsx` flows | Long-running; presumably terminated by SIGTERM / `claude daemon stop` | Throws on fatal; **must not** `process.exit()` (`bridgeMain.ts:2802`). Exits permanent vs transient by catching in the worker shell. |
| Session registry | `kind: 'interactive'` | `kind: 'daemon'` | `kind: 'daemon-worker'` |
| Crash recovery | User notices, re-runs `claude` | Presumably external (systemd/launchd) | Supervisor respawns with backoff unless `EXIT_CODE_PERMANENT` |
| Cron-task filter | All tasks except `permanent` (inferred) | N/A — delegates to workers | Cron worker uses `t => t.permanent` (`cronScheduler.ts:123-124`) |
| Kairos gate | Checked each startup | Checked once by supervisor | Child runs with `--assistant` flag to skip re-check (`main.tsx:1052-1056`) |

---

## 10. Integration points

External systems the daemon interacts with (by reference):

- **`daemon.json`** — supervisor-side config. Schema not in the
  extracted tree. Named at `bridgeMain.ts:2803` and
  `dialogLaunchers.tsx:69`.
- **`~/.claude/sessions/<pid>.json`** — PID registry. Populated by
  `registerSession()` once workers call into
  `concurrentSessions.ts` (or the supervisor spawns them with
  `CLAUDE_CODE_SESSION_KIND=daemon-worker`).
- **`scheduled_tasks.json`** — cron/scheduled task store, partitioned
  by `permanent` flag between daemon and non-daemon callers.
- **Bridge API** — `registerBridgeEnvironment`, `reconnectSession`,
  etc., same surface as the interactive `remote-control` bridge,
  just driven from `runBridgeHeadless`.
- **Upstream proxy** — `upstreamproxy.ts:13,139` factors retry into
  "supervisor restart retries".
- **Process spawning (supervisor → worker)** — spawn protocol is in
  the missing `daemon/main.ts`. From the CLI-arg shape
  (`cli.tsx:100`) it's `claude --daemon-worker <kind>` with env
  `CLAUDE_CODE_SESSION_KIND=daemon-worker` and presumably
  `CLAUDE_CODE_SESSION_NAME`, `CLAUDE_CODE_SESSION_LOG`,
  `CLAUDE_CODE_AGENT` (used by `registerSession()` at
  `concurrentSessions.ts:89-94`).
- **IPC channel** — implementation not visible. Used for (a) token
  hand-off (`getAccessToken`), (b) 401 refresh (`onAuth401`), (c)
  log forwarding (`log`), (d) lifecycle signals (`signal`).
- **`CLAUDE_CODE_MESSAGING_SOCKET`** — UDS inbox path
  (`concurrentSessions.ts:86-88`). Gated by `feature('UDS_INBOX')`.
  A plausible location for the supervisor↔worker IPC, but not
  confirmed here.

---

## 11. Worker-management complexity — flagged per task description

The task asked to flag "if worker management is complex". **Yes,
non-trivial.** Evidence:

1. **Error taxonomy is load-bearing and bespoke.** Permanent vs
   transient classification determines retry-vs-park policy, and
   the classification is spread across *call sites* (preflight
   checks in `runBridgeHeadless`, `fatalExit` flag in the loop,
   registration failure at 2911-2913). Each new worker kind needs
   to participate in this classification. The only guard is the
   worker kind's own author correctly throwing
   `BridgeHeadlessPermanentError` vs plain `Error`.
2. **Auth is shared across processes by callback.** `getAccessToken`
   crosses a process boundary every invocation. Any misbehaviour —
   stalls, failures, double-refresh — cascades across every worker
   the supervisor owns.
3. **At least three worker kinds exist** (remoteControl, assistant,
   cron) with different init requirements (sinks or not, configs
   or not, Kairos entitlement or not, permanent-task partition or
   not), yet they share one `workerRegistry.ts` dispatcher.
4. **State duplication with `--bg` sessions.** Both register in
   `~/.claude/sessions/`, both use `CLAUDE_CODE_SESSION_KIND`, but
   `--bg` has its own lifecycle (tmux-based) and `--continue` has
   to avoid both (`conversationRecovery.ts:487-489`). The surface
   area between "bg session" and "daemon worker" is finicky.
5. **Token-refresh has a v1/v2 split** inside the worker
   (`bridgeMain.ts:279-313`) — v1 updates the transport token
   directly, v2 calls `reconnectSession` to force server re-dispatch.
   CC-1263 is a named incident the v2 path exists to fix. This is
   subtle logic that lives inside the worker but only manifests
   because of the supervisor's 5h session lifetime expectations.

Worker management is **complex enough to warrant a dedicated
deep-dive task once the `daemon/` source is re-extracted.**

---

## 12. Follow-up recommendations

1. **Re-extract with `feature('DAEMON')` enabled** (or arrange
   out-of-band access to `src/daemon/`). Without it, the supervisor
   internals — the most interesting part of this subsystem — can't
   be analyzed.
2. **Re-extract with `feature('KAIROS')` enabled** too.
   `src/assistant/` has only 1 file; the rest is gated out.
   Assistant-mode is the most visible daemon consumer.
3. **Add a dedicated task for the `daemonMain` lifecycle** — spawn
   scheduling, backoff algorithm, graceful-shutdown sequence,
   subcommand set (start/stop/status/install). Currently no task
   on the list covers this.
4. **Add a task for `workerRegistry.ts` and the worker-kind
   dispatcher** — it's the policy layer for permanent-vs-transient
   exit, per-kind init, and log routing.
5. **Add a task for the daemon's `AuthManager`** — the IPC shape,
   refresh contention handling, and 401 propagation.
6. **Document the full worker-kind enumeration**. Comments name
   `remoteControl`, `cron worker`, `assistant`. There may be more.
   This should live alongside the daemon config schema.
7. **Cross-reference with task #10 (bridge/remote-control and
   template jobs)** — the bridge headless worker (§4 here) is the
   same code both tasks depend on. Ensure reports are consistent.
8. **Flag the WSL pruning gap** (`concurrentSessions.ts:194-201`):
   when running the daemon on Windows while viewers are on WSL (or
   vice versa), the prune logic sidesteps false positives by
   *skipping* pruning. This means stale daemon-worker PID files
   accumulate on WSL viewers over time. A dedicated daemon
   registry file could avoid the collision entirely.

---

## 13. Inventory of files read

| File | Purpose |
| --- | --- |
| `src/entrypoints/cli.tsx:80-180` | Worker + supervisor fast-path dispatch |
| `src/bridge/bridgeMain.ts:275-335` | Token-refresh scheduler, cleanup tracking, fatalExit |
| `src/bridge/bridgeMain.ts:2760-2965` | Headless worker entrypoint |
| `src/utils/concurrentSessions.ts` (full file) | Session-kind telemetry, PID registry |
| `src/utils/cronScheduler.ts` (first 130 lines) | `assistantMode`, cron worker's `permanent` filter |
| `src/utils/cronTasksLock.ts:30-45` | Daemon lockIdentity pattern |
| `src/utils/sinks.ts:5-9` | "daemon bypasses setup()" contract |
| `src/state/AppStateStore.ts:126-131` | Viewer-mode task-count event sourcing |
| `src/main.tsx` (selected): `80, 1049-1086, 2206, 2518, 2641-2645, 3286-3292, 3842` | Kairos gate, `--assistant` flag, daemon/assistant install UX |
| `src/components/StatusLine.tsx:30-33` | Statusline suppression in assistant mode |
| `src/hooks/useRemoteSession.ts:95-135, 175-185, 530-540` | Remote-session compaction/reconnect quirks, worker echo dedup |
| `src/bridge/replBridge.ts` (grep hits) | BridgeCoreParams shape, daemon-vs-REPL branches |
| `src/dialogLaunchers.tsx:60-80` | `claude assistant` empty-daemon.json install wizard |
| `src/utils/conversationRecovery.ts:485-495, 2620-2635` | `--continue` skip, stale-session warning for daemons |
| `src/upstreamproxy/upstreamproxy.ts:10-20, 135-145` | "supervisor restart retry" design |

Files **expected but missing**: `src/daemon/main.ts`,
`src/daemon/workerRegistry.ts`, most of `src/assistant/**`.
