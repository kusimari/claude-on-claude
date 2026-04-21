---
title: Claude Code CLI — Codebase Architecture Report
source: Wave 1 deep-dive reports (17 subsystems) synthesized via summaries/phase-{1..4}-summary.md and synthesis-plan.md
scope: Architecture narrative for the extracted `src/` tree of the Claude Code CLI
generated: 2026-04-21
---

# Claude Code CLI — Codebase Architecture Report

## 1. Executive Summary

Claude Code is a TypeScript terminal application — React/Ink UI, Bun-built, Anthropic-SDK-driven — whose architecture is shaped less by clean separation of concerns than by a set of load-bearing invariants that the code is carefully structured around. Seventeen subsystems organize into four interaction clusters: **Bootstrap** (setup, trust/auth, tools, MCP, state/config), **Request Cycle** (REPL → query → prompt → API → parsing → inner agent loop), **Lifecycle** (interrupts, persistence, cleanup), and **Background & Alternative Modes** (concurrent peers, daemon, bridge, templates). A thin ring of cross-cutting concerns — the trust boundary, prompt-cache discipline, abort tree, hooks surface, subagent scoping, and time-bounded async — threads through all four clusters and accounts for most of the apparent complexity.

### Architecture at a glance

The entry point (`src/entrypoints/cli.tsx`) dispatches on argv prefix *before* loading most of the CLI: daemon worker, daemon supervisor, bridge, templates, and `--bg`/`ps`/etc. each get a feature-gated fast path; everything else falls through to an interactive REPL (or headless `-p`). The interactive path funnels through `init() → setup() → showSetupScreens (trust gate) → first render → deferred prefetches → REPL mount → submit-gated background housekeeping`. Every long-lived peer registers a teardown in `cleanupRegistry`; every async phase in shutdown has a cap; every background timer is `.unref()`'d.

### Critical subsystems

- **`src/screens/REPL.tsx`** and **`src/query.ts`** (69KB) form the request-cycle hub: REPL owns the `AbortController` and `QueryGuard`, query.ts implements the per-iteration state machine with 10+ decision branches.
- **`src/utils/sessionStorage.ts`** (5105 LOC) is the most carefully-engineered subsystem examined: five persistence paths, file-size-gated load strategies, a 64KB tail-window invariant, and two in-memory splice mechanisms for compaction and snip.
- **`src/utils/gracefulShutdown.ts`** (529 LOC) centralizes exit: sync-first terminal cleanup before any `await`, 13-step pipeline with time-bounded phases, `EIO → SIGKILL` fallback.
- **`src/services/mcp/client.ts`** (3348 LOC), **`src/bridge/bridgeMain.ts`** (2999 LOC), and **`src/bridge/replBridge.ts`** (2406 LOC) are the largest peer-transport modules.

### Notable patterns

- **Init-bypass for fast start**: bridge and daemon workers skip `init.ts` and re-do only `enableConfigs`/`initSinks`/CWD/trust — trading duplication for boot latency.
- **Feature-gated dead-code elimination**: `feature('DAEMON')`, `feature('TEMPLATES')`, `feature('KAIROS')`, ant-only observability (`"external" === 'ant'`). The extracted snapshot omits `src/daemon/` and `src/cli/handlers/templateJobs.js` entirely — only the *contract* with them is analyzable.
- **Post-incident fingerprints everywhere**: CC-34 (session-id atomicity), gh-32712 (failsafe budget truncation), inc-3930 (tombstone OOM), ANT-344 (persistent-retry keep-alive stopgap), Bun sigaction pin (signal-exit v4 bug), gh-30217 (transcript_path cwd mismatch), and ~15 others each correspond to a specific comment-anchored invariant. This is the single strongest signal of how production-hardened the code is.
- **Prompt-cache discipline**: one boundary marker, byte-exact text preservation, sticky beta-header latches cleared only on explicit user actions (`/clear`, `/compact`). Broken cache → cost/latency cliff.

### Entry points for investigation

- Interactive path: `src/entrypoints/cli.tsx:main()` → `src/main.tsx:action()` → `src/interactiveHelpers.tsx:renderAndRun` → `src/screens/REPL.tsx`.
- Headless path: `src/cli/print.ts:runHeadless`.
- Per-turn loop: `src/query.ts:queryLoop` (primary), `src/query/transitions.ts` (state reasons — file not in snapshot; inferred).
- Shutdown: `src/utils/gracefulShutdown.ts:gracefulShutdown` (pipeline), `src/utils/cleanupRegistry.ts` (registry).

Remaining sections walk each cluster, surface the cross-cutting concerns, and collect flat reference material into appendices.

## 2. Runtime Topology

Claude Code runs as a main Node/Bun process plus a constellation of peers — subprocesses, long-lived workers, and external services — all coordinated through the main process's cleanup registry, AbortController tree, and IPC contracts. The diagram below shows the four process kinds (`interactive` / `bg` / `daemon` / `daemon-worker`) registered in `concurrentSessions.ts`, the external peers each spawns, and the transports that carry data between them.

```
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                       HOST PROCESS (Bun/Node, one per SessionKind)          │
 │                                                                             │
 │  ┌───────────────────────────────────────────────────────────────────────┐  │
 │  │  Entrypoint dispatch — src/entrypoints/cli.tsx                        │  │
 │  │  argv prefix → fast path:  --daemon | --daemon-worker <kind>          │  │
 │  │                             remote-control | new/list/reply          │  │
 │  │                             -p (headless) | (interactive REPL)        │  │
 │  └───────────────────────────────────────────────────────────────────────┘  │
 │     │                                                                       │
 │     ▼                                                                       │
 │  ┌──────────────────────┐   ┌──────────────────────┐                        │
 │  │  init() / setup()    │   │ init-bypass variants │  (bridge, daemon-      │
 │  │  main.tsx:1903-1935  │   │ narrow subset only:  │   worker — re-do only  │
 │  │  trust gate required │   │ enableConfigs +      │   what they need for   │
 │  │                      │   │ initSinks + cwd +    │   fast cold start)     │
 │  └──────────────────────┘   │ trust                │                        │
 │     │                       └──────────────────────┘                        │
 │     ▼                                                                       │
 │  ┌──────────────────────────────────────────────────────────────────────┐   │
 │  │  Ink render tree (interactive only) — src/screens/REPL.tsx           │   │
 │  │  ┌──────────┐  ┌──────────────┐  ┌────────────────┐  ┌────────────┐  │   │
 │  │  │ Messages │  │ PromptInput  │  │ PermissionUI   │  │ StatusBar  │  │   │
 │  │  └──────────┘  └──────────────┘  └────────────────┘  └────────────┘  │   │
 │  │   hosts: AbortController, QueryGuard, useCommandQueue,               │   │
 │  │          useScheduledTasks, useManageMCPConnections                  │   │
 │  └──────────────────────────────────────────────────────────────────────┘   │
 │     │                                                                       │
 │     ▼                                                                       │
 │  ┌──────────────────────────────────────────────────────────────────────┐   │
 │  │  query.ts → queryLoop (per-turn state machine, 10+ decision branches)│   │
 │  │  streams through services/api/claude.ts (withRetry + raw SDK)        │   │
 │  │  dispatches tools via StreamingToolExecutor                          │   │
 │  └──────────────────────────────────────────────────────────────────────┘   │
 │     │                                                                       │
 │     ▼                                                                       │
 │  ┌──────────────────────────────────────────────────────────────────────┐   │
 │  │  cleanupRegistry + gracefulShutdown (13-step sync-then-async exit)   │   │
 │  └──────────────────────────────────────────────────────────────────────┘   │
 └──────────────────────────┬───────────────┬────────────────┬─────────────────┘
                            │               │                │
             ┌──────────────┘               │                └──────────────┐
             │                              │                               │
             ▼                              ▼                               ▼
   ┌──────────────────┐          ┌──────────────────────┐         ┌──────────────────┐
   │ MCP subprocesses │          │ Bridge / Remote WS   │         │ Tool subprocesses│
   │ (stdio child)    │          │ (HTTP/SSE/WS peer)   │         │ (Bash, computer, │
   │ mcp/client.ts    │          │ bridge/*.ts          │         │  shell tasks)    │
   │ SIGINT→TERM→KILL │          │ v1 env-based         │         │ AbortSignal      │
   │ 100→500→600ms    │          │ v2 env-less SSE+CCR  │         │ propagation via  │
   │                  │          │                      │         │ toolUseContext   │
   └──────────────────┘          └──────────────────────┘         └──────────────────┘

 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                   CROSS-PROCESS COORDINATION (disk + UDS)                   │
 │  ~/.claude/sessions/<pid>.json    SessionKind registry (`claude ps`)        │
 │  ~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl   transcripts         │
 │  ~/.claude/history.jsonl           prompt history (locked, global)          │
 │  ~/.claude.json                    Config (write-through)                   │
 │  CLAUDE_CODE_MESSAGING_SOCKET      UDS inbox (bg handoff, daemon IPC)       │
 │  scheduled_tasks.json              cron (partitioned by permanent flag)     │
 └─────────────────────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                DAEMON TOPOLOGY (feature('DAEMON'), missing source)          │
 │                                                                             │
 │   claude daemon (supervisor, SessionKind=daemon)                            │
 │        │                                                                    │
 │        │ spawns children as:  claude --daemon-worker <kind>                 │
 │        │                                                                    │
 │        ├──▶ remoteControl worker (runBridgeHeadless — only analyzable body) │
 │        ├──▶ cron worker        (inferred from cronScheduler.ts:123-124)     │
 │        └──▶ assistant worker   (inferred from main.tsx:1052-1056, Kairos)   │
 │                                                                             │
 │   Shared: AuthManager IPC (getAccessToken / onAuth401 callbacks)            │
 │   Error taxonomy: BridgeHeadlessPermanentError → EXIT_CODE_PERMANENT → park │
 │                   plain Error → transient → respawn with backoff            │
 └─────────────────────────────────────────────────────────────────────────────┘
```

### Components

- **Host process** — single event loop; "concurrent" activity = overlapping async + `.unref()`'d intervals. Heavy CPU lives in subprocesses (MCP servers, Bash, analytics, bridge workers).
- **Entrypoint dispatch** (`src/entrypoints/cli.tsx`) — argv prefix determines fast path *before* most of the CLI loads. Daemon supervisor, daemon worker, bridge, templates, and `--bg`/`ps` each short-circuit full init.
- **Ink render tree** (`src/screens/REPL.tsx`) — interactive-only. Owns the per-turn `AbortController`, `QueryGuard`, command queue, cron mount, and MCP connection manager.
- **Request-cycle core** (`query.ts` + `services/api/claude.ts`) — streaming pipeline with withRetry, tool executor, prompt-cache discipline.
- **Cleanup + shutdown** (`utils/cleanupRegistry.ts` + `utils/gracefulShutdown.ts`) — 43 registering files, 13-step exit pipeline with time-bounded phases.
- **MCP subprocesses** (`src/services/mcp/client.ts`) — stdio children with SIGINT→SIGTERM→SIGKILL escalation and 600ms absolute failsafe; remote MCP via HTTP/SSE/WS.
- **Bridge / Remote WS** (`src/bridge/*.ts`, ~12.6k LOC across 31 files) — three entrypoints (standalone, REPL-hosted, daemon-worker) converge on `runBridgeLoop` (v1) or v2 SSE+CCR transport.
- **Tool subprocesses** — Bash, computerUse, shell tasks; `AbortSignal` propagated via `toolUseContext`; subprocess-tree killing is each tool's responsibility.
- **Cross-process coordination** — PID registry + JSONL transcripts + UDS inbox + `~/.claude.json` + `scheduled_tasks.json`. No central lock manager — per-resource locking (history.jsonl global lock; transcripts per-session no-contention).
- **Daemon topology** (feature-gated, source missing) — supervisor + workers pattern; workers inherit narrow IPC contract defined by `HeadlessBridgeOpts`.

### Process-kind matrix

| SessionKind    | Entrypoint                               | Ink tree | init.ts | Trust gate   | Registers in PID file |
|----------------|------------------------------------------|----------|---------|--------------|-----------------------|
| `interactive`  | `cli.tsx` → REPL                         | yes      | full    | interactive  | yes                   |
| `bg`           | `cli.tsx --bg`                           | no       | partial | inherited    | yes                   |
| `daemon`       | `cli.tsx --daemon` (supervisor)          | no       | partial | supervisor   | yes                   |
| `daemon-worker`| `cli.tsx --daemon-worker <kind>`         | no       | bypass  | inherited    | yes                   |

All four kinds register with `cleanupRegistry` and honor `gracefulShutdown`; only `interactive` mounts the Ink tree and the per-turn AbortController tree.

## 3. Bootstrap (Cluster A)

Cluster A — setup, trust/auth, tool-system init, MCP bootstrap, State/Config/Settings — establishes the pre-REPL readiness contract. The five members share a strict ordering, a single trust gate, and produce the inputs (trusted env, tool pool, MCP pool, session state) consumed by every later cluster. Skip or reorder any phase and downstream code reads undefined state.

### 3.1 `setup()` phases

`setup()` (`src/setup.ts`, ~480 LOC) is the interactive-session environment gate. It runs after Commander's `preAction` calls `init()` (which handles all subcommands) and before `showSetupScreens()` (the trust gate). Call site: `src/main.tsx:1903-1935`.

**Twelve phases (A–L)**:

```
A. Node-version gate          (setup.ts:70-79)    — hard requirement
B. UDS messaging start        (:89-102)           — await so SessionStart hooks inherit socket
C. Teammate snapshot                              — capture current team state
D. Terminal restore                               — restore terminal modes from any prior session
E. cwd + hooks snapshot                           — setCwd() MUST precede captureHooksConfigSnapshot()
F. Worktree creation          (:176-285, ~110 LOC)— may chdir mid-function; clears memory-file caches;
                                                    re-captures hooks from new cwd
G. Background prefetches      (:287-330)          — UDS, watchers, sinks, API key helper
H. ant/attribution hooks                          — deferred via setImmediate (prevents main-tsx cycle)
I. initSinks() + tengu_started (:371-381)         — earliest reliable telemetry beacon
J. Release-notes prefetch                         — cached for later banner
K. Sandbox safety             (:396-442)          — bypass-permissions enforcement
L. Prior-session exit replay  (:449-476)          — tengu_exit from last crash replayed into analytics
```

**Ordering invariants**:
- `setCwd()` → `captureHooksConfigSnapshot()` — snapshot must reflect final cwd after worktree.
- `await startUdsMessaging` before anything that fires `SessionStart` hooks — hooks inherit `CLAUDE_CODE_MESSAGING_SOCKET` via env.
- `tengu_started` (Phase I) is the earliest reliable beacon — moving it later loses telemetry for early crashes.

**`--bare` mode** disables UDS, session memory, plugins prefetch, attribution, release notes — keeps node check, worktree, sandbox safety, sinks. Used for scripted/non-interactive invocations.

**Worktree (Phase F)** is special — ~110 LOC mutating `cwd` / `originalCwd` / `projectRoot`, clearing memory-file caches, re-capturing the hooks snapshot. Prior to the gh-30217 fix, post-worktree `transcript_path` could disagree with cwd.

**Parallelism** — `setup()` runs in parallel with `getCommands()` + `getAgentDefinitionsWithOverrides()` unless worktree is enabled (worktree serializes because it mutates cwd).

### 3.2 Trust boundary + auth cascades

`showSetupScreens()` (`src/interactiveHelpers.tsx:104-298`) is the **single trust gate**. Interactive-only; CI / print mode / `--bare` short-circuit before it.

**Fixed dialog order** (each blocks on prior acceptance):
```
Onboarding → TrustDialog → MCP-json approvals → CLAUDE.md external-includes
  → Grove policy → ApproveApiKey → BypassPermissions → AutoMode
  → DevChannels → ChromeOnboarding
```

**Trust persistence** (`utils/config.ts:697-743`):
- **Session flag** (STATE) + **per-directory disk** (`config.projects[path].hasTrustDialogAccepted`)
- **Home directory** is session-only — never persisted. Prevents auto-trust of all cloned repos.
- **`computeTrustDialogAccepted()`** walks parent directories; any trusted ancestor propagates.
- **Memoize-only-true** — `false` re-checked so mid-session acceptance picks up.

**What trust unlocks, in order** (`interactiveHelpers.tsx:144-190`):
1. `setSessionTrustAccepted(true)`
2. `resetGrowthBook()` + `initializeGrowthBook()` (auth'd flag evaluation)
3. `applyConfigEnvironmentVariables()` (full env apply)
4. `setImmediate(initializeTelemetryAfterTrust)` (telemetry init)
5. OAuth helpers / API-key helper invocation
6. `updateGithubRepoPathMapping`
7. Managed-env apply

**Two parallel auth cascades**:

- **OAuth** (`utils/auth.ts:153-206`, `getAuthTokenSource`): env → FD → apiKeyHelper → secureStorage. PKCE flow with localhost callback listener. Enterprise `forceLoginOrgUUID` enforced against *uncached* server profile (auth.ts:1923-2000) so stale tokens across restarts can't bypass.
- **API key** (`utils/auth.ts:226-348`, `getAnthropicApiKeyWithSource`): separate cascade; `--bare` bypasses most steps.

**401 / refresh discipline** (auth.ts:1360-1562):
- In-flight refresh deduped by token value — multiple 401s collapse to one refresh.
- Filesystem lock on `~/.claude/` serializes refreshes across concurrent processes.
- Scope-expansion refresh for step-up `insufficient_scope`.

**Decline paths** call `gracefulShutdownSync(0|1)` — Grove exit-0 (policy-driven), TrustDialog decline exit-1.

### 3.3 Tool pool assembly

Five-stage pipeline:

```
getAllBaseTools()      ← all compiled-in tools
   ↓
getTools(context)      ← filters: simple mode, deny rules, REPL flags, isEnabled gates
   ↓
assembleToolPool(ctx, mcpTools)   ← partitions into [...builtIn.sort, ...mcp.sort]
   ↓
useMergedTools (with initialTools, coordinator filter)
   ↓
getToolForApi()        ← zodToJsonSchema + cache; strips swarm-only fields
```

**Built-in-prefix ordering is prompt-cache-load-bearing** (§7.2): `assembleToolPool` (`tools.ts:345`) and `mergeAndFilterTools` (`toolPool.ts:55`) both re-partition `[...builtIn.sort, ...mcp.sort]`. Reorder between requests → prompt cache busts → cost/latency cliff.

**8-step permission pipeline** (`permissions.ts:1158`):
```
1a deny → 1b ask → 1c tool.checkPermissions → 1d tool-deny
   → 1e requiresUserInteraction → 1f content-ask → 1g safetyCheck
   → 2a bypass → 2b allow → 2c ask-default
```

- `--dangerously-skip-permissions` **still runs 1g safetyCheck** — never total bypass.
- Deny rules strip tools from the visible catalog **only if tool-scoped**; input-scoped denies stay in the pool.

**Permission modes**: `default`, `plan`, `acceptEdits`, `bypassPermissions`, `auto`, `dontAsk`.
- **Auto mode** strips dangerous Bash/PowerShell/Task permissions into `strippedDangerousRules` for restore on exit; uses `TRANSCRIPT_CLASSIFIER` with 2s grace race to promote `ask` → `allow`.
- Transitions: `permissionSetup.ts:597` handles mode changes.

**Subagent tool filtering** (`agentToolUtils.ts:70` `filterToolsForAgent`) — layered denies:
- `ALL_AGENT_DISALLOWED` (never)
- `CUSTOM_AGENT_DISALLOWED`
- `ASYNC_AGENT_ALLOWED` (allowlist, excludes MCP)
- `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` (teammate-specific: `SendMessage`, `TaskUpdate`)

Subagents force `shouldAvoidPermissionPrompts=true` and inherit parent's mode / additionalDirectories (§7.5).

**Simple mode** (`CLAUDE_CODE_SIMPLE=1`) = `[Bash, Read, Edit]` only.
**Coordinator mode** (`feature('COORDINATOR_MODE')`) narrows pool to `COORDINATOR_MODE_ALLOWED_TOOLS` + PR activity subscription suffix.

### 3.4 MCP pool bootstrap

MCP = plugin / extensibility vector. 0–100+ servers per session over **7 transports**: `stdio`, `sse`, `http`, `ws`, `sse-ide`, `ws-ide`, `sdk`, `claudeai-proxy`.

**Config discovery — 7 scopes, merged + deduped by signature**:
```
enterprise (exclusive if present)
  > user
  > project (.mcp.json, parent walk)
  > local
  > dynamic (--mcp-config / plugin)
  > claudeai
  > managed
```
Signatures: stdio JSON-command or URL-unwrapped (`mcp/config.ts:202-212`).

**Connection** (`mcp/client.ts:595-1647` `connectToServer`):
- Memoized by `${name}-${JSON.stringify(serverRef)}`
- Per-transport setup (`:619-865`)
- In-process variant for Chrome / Computer Use (`:905-943`, `InProcessTransport.ts`)
- `onclose` (`:1221-1402`) clears cache + re-fetch caches
- Reconnect backoff 1s→30s, `MAX_RECONNECT_ATTEMPTS=5`, remote only (stdio dies → no auto-reconnect)

**Tool registration**:
- Qualified names `mcp__{server}__{tool}` (except `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`)
- `inputJSONSchema` passes through **verbatim** (no Zod conversion — MCP is the only source of non-Zod tool schemas)
- MCP hints mapped to Tool methods: `readOnlyHint`, `destructiveHint`, `openWorldHint`

**OAuth** (`mcp/auth.ts:1376` `ClaudeAuthProvider`):
- PKCE + DCR + CIMD (client metadata URL) + XAA (cross-app ID token exchange)
- Step-up: 403 `insufficient_scope` → force PKCE
- 15-minute `needs-auth` cache TTL prevents probe spam
- `tokens()` called 30–40×/sec → cache hit-rate critical

**Two-phase load** in `useManageMCPConnections` (`:858-1024`):
- Regular configs concurrent with claude.ai fetch
- Updates batched every 16ms via `pendingUpdatesRef` (prevents render thrash)

**Timeouts** (constants):
- `MCP_TIMEOUT` 30s (connect)
- `MCP_REQUEST_TIMEOUT_MS` 60s
- `MCP_TOOL_TIMEOUT` ~27.8h (effectively unbounded for long tools)
- `MCP_AUTH_CACHE_TTL_MS` 15min
- `MCP_BATCH_FLUSH_MS` 16

**Shutdown escalation** (§5.5): stdio SIGINT 100ms → SIGTERM 500ms → SIGKILL 600ms.

**Open question**: no `.mcp.json` file watcher detected — config changes require `/mcp reload`.

### 3.5 State / Config / Settings three-layer model

Three distinct persistence layers, each with different semantics:

**STATE** — `src/bootstrap/state.ts`:
- Process-scoped module singleton
- **No persistence** — clean state each process
- Bare getters / setters (type at `:45-257`)
- **Bootstrap-isolation eslint-enforced** — nothing outside `bootstrap/` may import; forces explicit dependency flow.
- Holds: sessionId, projectRoot, originalCwd, mainLoopModelOverride, trust flag, cost tracking, latch states.

**Config** — `~/.claude.json` via `utils/config.ts`:
- **Write-through cache** + file-watcher freshness (`:1007-1086`)
- **File lock on save** (`config.json.lock`, `:1153-1329`) — re-reads under lock before writing
- **Auth-loss guard** (GH #3117): refuses save if `oauthAccount` present in cache but missing on disk
- **Timestamped backups** — min 60s apart, 5 kept
- **Migrations** (`:912-960`) — `migrationVersion` gate prevents N× saveGlobalConfig on startup

**Settings** — `utils/settings/*`:
- **5-source Zod-validated merge** — `policy > flag > local > project > user`
- **Read-only at runtime** (mutations go through `applySettingsChange`, not direct writes)
- **Array concat+uniq** via `settingsMergeCustomizer` (`:538`)
- **`hasAutoModeOptIn`** intentionally excludes `projectSettings` (`:882-910`) to block RCE — a malicious repo can't opt itself into auto mode.

**MDM (policy source) mechanics**:
- **macOS**: plist
- **Windows**: registry HKLM > managed-file > HKCU
- **Linux**: drop-in dir `/etc/claude-code/managed-settings.d/*.json` (alphabetical)
- Subprocess spawned at `main.tsx` top-level, awaited before first settings access (`main.tsx:914`).

**Two env-apply paths**:
- **`applySafeConfigEnvironmentVariables()`** — **pre-trust**; `userSettings` + `flagSettings` + `policySettings` only; `SAFE_ENV_VARS` allowlist stripping `LD_PRELOAD` / `PATH`.
- **`applyConfigEnvironmentVariables()`** — **post-trust**; full apply.

**Bootstrap ordering** (the whole cluster in one line):
```
enableConfigs() → await MDM → first settings access → showSetupScreens (trust gate)
  → applyConfigEnvironmentVariables → resetGrowthBook+initializeGrowthBook
  → telemetry init → tool pool assembled → MCP pool kicked off → first render
```

**Beta-header latches** (`afkModeHeaderLatched`, `promptCache1hAllowlistLatched`, ...) live on STATE but are structurally part of Settings discipline. Sticky-on through session — must not flip mid-session (§7.2). Cleared only by `/clear` + `/compact`.

> **Cross-refs**: trust boundary §7.1, prompt cache §7.2, subagent scoping §7.5, hooks §7.4, post-incident fingerprints §7.7 (gh-30217, #3117).

## 4. Request Cycle (Cluster B)

The request cycle is the tightest loop in the system — user prompt → query → API → stream → tool → observation → next turn. Six subsystems (REPL input, query orchestration, prompt construction, API streaming, response parsing, inner agent loop) interlock through linear data flow with the inner agent loop (`queryLoop`) as the decision hub. Every hot-path invariant in this cluster has a post-incident fingerprint — this is where most production scars live.

### 4.1 REPL input + QueryGuard

The `REPL` component (`src/screens/REPL.tsx:324`, ~5000 LOC) owns interactive UI state: message list, tool state, command queue, mode, dialogs — all accessed via `useAppState(selector)` (Zustand-like). It hosts the per-turn `AbortController` and `QueryGuard`.

**QueryGuard** (`REPL.tsx:897-900`) is the single serialization point: `idle → queued → running → completing → idle`. Enforces one-query-at-a-time; a second prompt submitted mid-turn resolves via `pendingQueueRef` when the current turn returns `Terminal`.

**onQuery path** (`REPL.tsx:1156-1289`):
1. Text → `processUserInput()` → slash-command extraction
2. `query()` async generator invoked
3. For-await drains into `useAppState.messages` + transcript writer (100ms flush)
4. Abort wired to `abortController.abort('interrupt' | 'escape' | 'background')`

**Slash-command split**: pre-loop commands (locally resolved — `/clear`, `/compact`, `/help`) handled before `query()` is even called; in-loop commands (need model — `/review`, `/explain`) dispatched through `processSlashCommand` and run inside the loop. `SlashCommand` attachments route differently from plain text input.

**Input modes** (`src/components/PromptInput.tsx` + `useInputMode`): normal, paste-buffer (bracketed paste), command-palette (Ctrl+K), image-paste, history-cycle. Paste handling at `REPL.tsx:3014-3098`.

### 4.2 Context preparation (5-way compaction stack)

Before the API is called, the message history passes through a strict pipeline (`query.ts:365-548`) — the order is load-bearing and documented in comments at `query.ts:396-399`:

```
getMessagesAfterCompactBoundary
  ↓ (cut at compact_boundary parent anchor)
applyToolResultBudget
  ↓ (per-tool-result truncation)
snipCompactIfNeeded         ← snip before microcompact
  ↓
microcompact
  ↓
contextCollapse.applyCollapsesIfNeeded   ← collapse before autocompact
  ↓
autocompact
  ↓
[reactive_compact trigger reserved for 413 recovery — not pre-API]
  ↓
hard blocking_limit check
```

**Five interacting mechanisms**:
1. **Snip** — path-compression of user-selected `/snip` ranges via `snipMetadata.removedUuids` (sessionStorage.ts:1982-2039).
2. **Microcompact** — small in-turn summarization of low-value tool results.
3. **Context-collapse** — ordered `marble-origami-commit` log + `marble-origami-snapshot` last-wins for compressing large tool output.
4. **Autocompact** — full-session summary into `compact_boundary` when token budget exceeds threshold.
5. **Reactive-compact** — owned by 413/max_output_tokens recovery; `hasAttemptedReactiveCompact` flag preserved across stop-hook retries (death-spiral guard at `query.ts:1292-1297`).

Each mechanism has ordering invariants traced to specific incidents; reordering any pair breaks a known recovery path.

### 4.3 Prompt construction + cache boundary

`getSystemPrompt()` (`src/constants/prompts.ts:444-577`) assembles the system prompt in **two regions** separated by the single marker `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` (`prompts.ts:114`):

- **Prefix (cache-stable)**: tool descriptions, CLAUDE.md contents, agent definitions. Ordered fixed — any reorder busts cache.
- **Suffix (per-session)**: dynamic context, session-specific flags.

`splitSysPromptPrefix()` (`src/utils/api.ts:321-435`) partitions content blocks at the boundary and applies `cache_control: { type: 'ephemeral', ttl: '1h' | '5m' }` to the prefix. TTL selection is model + flag dependent.

**User context prepend** (`query.ts:660`, `prependUserContext`) injects directory listing, git status, recent file changes, memory — byte-stably, after system prompt, before turn messages. CLAUDE.md files (project / user / enterprise) are read at `startRelevantMemoryPrefetch` and merged in.

**Beta-header latches** — sticky-on through session, cleared only by `/clear` + `/compact`:
- `afkModeHeaderLatched`
- `promptCache1hAllowlistLatched`
- Other feature-gated beta headers

See §7.2 for full prompt-cache discipline discussion.

### 4.4 API streaming + fallback

`queryModel()` (`src/services/api/claude.ts:1017`, within a 3419-LOC file) is the streaming async generator. It opens a **raw stream** via `.withResponse()` — not `BetaMessageStream` — to avoid O(n²) `partialParse` on tool inputs (`claude.ts:1818-1836`).

**Idle watchdog stack** (`claude.ts:1868-1929`):
- `STREAM_IDLE_TIMEOUT_MS = 90s` — aborts silent connections.
- `STALL_THRESHOLD_MS = 30s` — emits `tengu_streaming_stall` telemetry (no abort).
- Both distinct from SDK request timeout (which only covers initial fetch).
- Abort-vs-timeout distinction (`:2434-2462`): `signal.aborted === true` → real user ESC; `false` → rethrow as `APIConnectionTimeoutError` for retry.

**Retry** (`withRetry.ts`, 822 LOC): exponential backoff with jitter, respects `Retry-After`, splits retryable (5xx, 408, 429, connection-timeout) from non-retryable (4xx, refusal, context-window). `FallbackTriggeredError` (model-under-load after `MAX_529_RETRIES=3`) bubbles to the outer query loop. Full state machine in Appendix A.

**Non-streaming fallback** (`claude.ts:2404-2569`, triggered on zero-event / incomplete stream): `executeNonStreamingRequest()` with per-attempt timeout 120s (remote) or 300s (local). Escape hatch `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` exists to avoid double-execution (inc-4258). `404` stream-creation triggers its own fallback at `:2612-2646`.

**Provider dispatch** (`services/api/client.ts`) — firstParty, Bedrock, Vertex, CCR, grove — handles base URL, header injection, auth source, region-specific behavior. `normalizeModelStringForAPI` (`claude.ts:867`) rewrites model names per provider; `CLIENT_REQUEST_ID_HEADER` is firstParty-only.

### 4.5 Per-block parsing + tool dispatch

Response events are consumed by a per-block streaming accumulator inside `queryModel()` (event switch at `claude.ts:1940-2304`). The critical invariant:

> **One `AssistantMessage` yielded per `content_block_stop`** — not per message. The consumer stitches blocks.

**Direct mutation invariant** (`claude.ts:2236-2248`): `lastMsg.message.usage = usage` — **not** replacement. The transcript write queue holds a reference; serialization runs on a 100ms flush interval. Replacing the object decouples the queued reference and drops writes. Load-bearing.

`normalizeContentFromAPI()` (`utils/messages.ts:2651-2751`):
- `tool_use` inputs parsed with `safeParseJSON` + `{}` fallback
- `text` preserved byte-exact (cache discipline)
- `thinking` passthrough preserves signature (model-bound)
- Known hole: recursive stringified JSON (`messages.ts:2670-2674`)

**`StreamingToolExecutor`** (`src/services/tools/StreamingToolExecutor.ts`, 530 LOC) dispatches tools **mid-stream** at `content_block_stop`, before the message completes:
- Concurrent-safe tools (Read, WebFetch) run in parallel.
- Non-concurrent-safe tools (Bash, Edit) block-queue for exclusive access.
- `siblingAbortController` cascades Bash errors only (Read/WebFetch independent — failures don't poison siblings).
- TrackedTool state machine handles discard/rebuild on streaming fallback.

**Stop-reason mapping** is subordinate to content presence:
- `end_turn` → exit if no tools
- `tool_use` → **not read** (content-block presence is authoritative; `stop_reason` documented-unreliable)
- `max_tokens` + `model_context_window_exceeded` → reuse max-output-tokens recovery
- `refusal` → synthesized AUP message via `getErrorMessageIfRefusal`

**Three adapters normalize streams in/out** and must stay in sync: VCR (`src/services/vcr.ts`), CCR (`src/services/ccrClient.ts`), Remote SDK (`src/remote/sdkMessageAdapter.ts`).

### 4.6 Decision hub + stop-hook handling

`queryLoop()` (`query.ts:241-1728`) is the state machine. Each iteration rebuilds `State` (10 fields, `query.ts:204-217`) at every `continue` site — only `toolUseContext` mutates mid-iteration. `transition` field gates one-shot recovery paths.

**`needsFollowUp`** (`query.ts:552-557`) is the sole continue signal — flipped true when any `tool_use` block arrives (`query.ts:829-835`). If false, the loop returns `Terminal`.

**7 Continue reasons** (from `src/query/transitions.ts` — file not in snapshot; inferred from usages):
`collapse_drain_retry`, `reactive_compact_retry`, `max_output_tokens_escalate`, `max_output_tokens_recovery`, `stop_hook_blocking`, `token_budget_continuation`, `next_turn`.

**12 Terminal reasons** (selection): `completed`, `aborted_streaming`, `aborted_tools`, `blocking_limit`, `image_error`, `model_error`, `prompt_too_long`, `stop_hook_prevented`, `hook_stopped`, `max_turns`, plus others. Full matrix in Appendix B.

**10+ decision branches at iteration end** (`query.ts:1062-1358`): collapse-drain retry (one-shot, `:1089-1117`), reactive-compact retry (`:1119-1175`), max-tokens escalate/resume (`:1199-1252`), stop-hook preventContinuation/blocking (`:1267-1306`), token-budget nudge, abort paths, maxTurns, hook-stopped, normal next-turn.

**Stop hooks** (`src/query/stopHooks.ts:65-472`): `executeStopHooks`, `executeStopFailureHooks`, teammate `TaskCompleted` / `TeammateIdle` (`:335-452`). Can `preventContinuation` or inject `blockingErrors` for retry.

**Withheld errors** (`query.ts:712-740`): 413 / media-not-supported / max_output_tokens surfaced during streaming are held back so SDK callers don't see recoverable intermediates — replayed only after `needsFollowUp === false`.

**Permission + hook interleaving** (`toolExecution.ts:921-1103`): hooks can override permissions, update input, stop tool, retry on `PermissionDenied`. The quadruple of state (`hookPermissionResult`, `hookUpdatedInput`, `shouldPreventContinuation`, `stopReason`) is tangled but necessary. MCP tool post-hook ordering differs because hooks rewrite output (§7.4).

**Injection seams** — `src/query/deps.ts`, `src/query/config.ts` — 4 `QueryDeps` (`callModel`, `microcompact`, `autocompact`, `uuid`) + immutable `QueryConfig` per session. Tests inject without spy-per-module.

> **Cross-refs**: abort tree is §7.3, prompt cache is §7.2, hooks are §7.4, time-bounds are §7.6, post-incident fingerprints are §7.7.

## 5. Lifecycle (Cluster C)

Cluster C covers the session end-of-life: three subsystems — interruption handling (13), session persistence (14), cleanup/exit (15) — share a choreography where signal cascades through the abort tree, retries/fallbacks decide whether work continues, persistence flushes all queued state, and the shutdown pipeline tears everything down under time-bound. Persistence is also active throughout the lifetime, but its cleanup-phase handshake is where the tightest invariants live.

### 5.1 Cancellation taxonomy

Three non-overlapping layers (cross-ref §7.3 for the full trust-boundary diagram):

**Layer 1 — OS signals** (`src/utils/gracefulShutdown.ts:237-334`):
- `SIGINT` → `gracefulShutdown(0)` (skipped in print mode — print handles its own SIGINT → query abort → shutdown at `src/cli/print.ts:1027-1034`)
- `SIGTERM` → exit 143
- `SIGHUP` → exit 129 (non-Windows)
- `uncaughtException` + `unhandledRejection` → logged but non-fatal; funnel to `void gracefulShutdown()`

**Platform workarounds**:
- **Bun sigaction pin** (`gracefulShutdown.ts:254`): `onExit(() => {})` no-op subscriber prevents signal-exit v4's `removeListener` from resetting kernel sigactions mid-session (silent-SIGTERM-no-op bug).
- **macOS TTY orphan detect** (`gracefulShutdown.ts:281-296`): 30s `.unref()`'d interval polls `stdout.writable && stdin.readable` because macOS revokes TTY fds without SIGHUP on force-close.

**Layer 2 — In-process `AbortController` tree** (`src/utils/abortController.ts:16-22`, `:68-99`):
- `createAbortController` with `maxListeners = 50`
- `createChildAbortController` with `WeakRef` to avoid memory pinning
- `combinedAbortSignal.ts:15-47` uses plain `setTimeout` instead of Bun's `AbortSignal.timeout` (2.4KB native-memory leak per timeout).

**Semantic abort reasons** dispatched via `signal.reason`:
- `'user-cancel'` (Esc / Ctrl+C) — abandon
- `'background'` (Ctrl+B) — handoff to bg session
- `'interrupt'` (`'now'`-priority queued command) — requeue

`isAbortError` triple-check (`errors.ts:27-33`): `AbortError | APIUserAbortError | name === 'AbortError'` — SDK class names minified in production.

**User-interrupt priority** (`src/hooks/useCancelRequest.ts:87-115`):
1. If permission prompt open → `toolUseConfirmQueue[0]?.onAbort()`
2. Else if remote mode → `activeRemote.cancelRequest()` (routes through CCR socket, not local abort)
3. Else → `abortController.abort('user-cancel')`
4. Two-press within window → kill-agents cascade (`:225-273`)

**Ctrl+B backgrounding** (`REPL.tsx:2528`): emits `'background'` reason, `useSessionBackgrounding` handoff to bg session kind.

### 5.2 Retry / fallback (`withRetry.ts`)

`withRetry.ts` (822 LOC) is the provider-agnostic retry state machine. It wraps every API call and handles 429/529/401/403/connection errors uniformly while delegating provider-specific auth recovery through callbacks.

**Core loop** (`:170-517`):
- Exponential backoff with jitter, respects `Retry-After` header
- Header check at loop top: `signal?.aborted` → throw `AbortError`
- `sleep(delayMs, signal, { abortError })` aborts mid-sleep

**Triage paths**:
- **Foreground 529** (`:62-82`): only retried when source is in `FOREGROUND_529_RETRY_SOURCES`. Background 529 retry was dropped — foreground-only to prevent cascade amplification. After `MAX_529_RETRIES=3`, throws `FallbackTriggeredError` to outer query loop (which picks up reactive-compact / Opus-fallback).
- **Persistent mode** (`:477-506`): `CLAUDE_CODE_UNATTENDED_RETRY=1` enables 30s yields with heartbeat; ANT-344 flags a TODO for a dedicated keep-alive channel.
- **Fast-mode cooldown** (`:267-314`): separate state in `utils/fastMode.ts`; retry triggers `triggerFastModeCooldown`.
- **Opus → fallback chain** (`:326-365`): model-under-load escalates through a model chain rather than bailing.
- **Context-overflow self-heal** (`:550-595`): 413 / `model_context_window_exceeded` trigger in-loop compaction rather than failing.

**Provider-specific auth recovery** (`:631-694`) — Bedrock, Vertex, OAuth, API-key each plug in via clear-cache callbacks so `withRetry` stays provider-agnostic.

**ConfigParseError recovery** (`src/entrypoints/init.ts:215-236`): invalid-config dialog lets the user fix/reset without hard-failing init.

### 5.3 Persistence write path

Per-session JSONL append-only log at `~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl` (files `0o600`, dirs `0o700`). 18+ discriminated `Entry` types in `Project.appendEntry` (`sessionStorage.ts:1128-1265`). Full type list in Appendix F.

**Write pipeline** (`sessionStorage.ts:618-686`):
- Per-file batched queue with `FLUSH_INTERVAL_MS = 100ms` (`10ms` for CCR v2 / session ingress)
- `MAX_CHUNK_BYTES = 100MB` safety cap per flush
- `pendingEntries` buffer pre-materialization — metadata-only orphan files don't leak if queue dies before any `message` entry writes.

**Per-file atomicity**:
- `materializeSessionFile` (`:976-991`) creates the file lazily on first real entry
- `insertMessageChain` (`:993-1083`) stamps `parentUuid` so downstream `walkChainBeforeParse` can do dead-fork elimination via the `{"parentUuid":` first-key invariant.

**64KB tail-window invariant** (the single most important persistence invariant):
- Pickers read **only the last 64KB** of each session file (`LITE_READ_BUF_SIZE`) for fast listing.
- `reAppendSessionMetadata` (`:721-839`) re-writes `customTitle` / `tag` / `gitBranch` / PR / agent fields at exit and on resume so they stay in the window.
- **`ai-title` intentionally never re-appended** — user renames via `customTitle` take precedence.

**Tombstone (logical deletion)** — `removeMessageByUuid` (`:871-951`):
- **Fast path**: 64KB tail scan if target is recent.
- **Slow path**: full-file rewrite capped at 50MB (inc-3930 OOM fix).

**Three persistence paths converge on the same queue**:
1. Local JSONL (default)
2. v1 Session Ingress (`persistToRemote` → `appendSessionLog`)
3. CCR v2 internal events (`setInternalEventWriter`)

Plus global prompt history at `~/.claude/history.jsonl` with `lock(historyPath)` (`history.ts:292-327`) — the only transcript-adjacent file that needs cross-process locking.

**Preservation mechanisms without rewriting JSONL**:
- `applyPreservedSegmentRelinks` (`:1839-1956`) — in-memory splice; zeroes usage tokens on preserved messages to prevent auto-compact spiral on resume.
- `applySnipRemovals` (`:1982-2039`) — `snipMetadata.removedUuids` drives path-compression.
- `compact_boundary` + `preservedSegment` writes for compaction flows.
- `marble-origami-commit` (ordered log) + `marble-origami-snapshot` (last-wins) for context-collapse.

### 5.4 Persistence load / resume / fork

**Load threshold split** — `SKIP_PRECOMPACT_THRESHOLD = 5MB`:
- Small files → parse with `parseJSONL` (all in memory).
- Large files → `readTranscriptForLoad` (`sessionStoragePortable.ts:717-793`) — chunked forward scan with attr-snap fd-level skip + boundary truncate.

**`walkChainBeforeParse`** (`sessionStorage.ts:3306-3466`): dead-fork elimination relies on `{"parentUuid":` being the JSON's first key. This is why `insertMessageChain` stamps `parentUuid` explicitly — reorder it and the walker stops working.

**Load pipeline** (`:3472-3813`):
1. `scanPreBoundaryMetadata` (`:3157-3224`) — extract metadata ahead of `compact_boundary`
2. Parse entries
3. `buildConversationChain` (`:2069-2094`) — stitch linked list via `parentUuid`
4. `checkResumeConsistency` (`:2224-2243`) — sanity checks

**Resume modes** (from `src/utils/conversationRecovery.ts:456-597` `loadConversationForResume`):
- `--continue` — most recent session in current cwd
- `--resume <id>` — by session ID
- `--resume <path>` — by file path
- `--resume <title>` — by customTitle match
- `--fork-session` — fresh sessionId, seeds `recordContentReplacement`, strips `worktreeSession` to avoid double-ownership (adamr-20260320 fix)

**Resume → bootstrap handshake**:
- `adoptResumedSessionFile` (`:1530-1534`) — rehydrate session
- `hydrateRemoteSession` (`:1587-1622`), `hydrateFromCCRv2InternalEvents` (`:1632-1723`) — bring remote state inline
- `processSessionStartHooks('resume', …)` bridges to bootstrap (§3)
- `processResumedConversation` (`src/utils/sessionRestore.ts:409-551`) — full post-load reconciliation

**Session-ID atomicity** (CC-34 fix) — `switchSession(sid, projectDir)` is atomic; prevents `transcript_path` / cwd mismatch (gh-30217).

**Worktree integration**:
- `worktree-state` entry is tri-state (`undefined` / `null` / object)
- `restoreWorktreeForResume` chdirs on resume so downstream reads the right path
- Fork strips `worktreeSession` to avoid double-ownership

**Ingress failure handling**:
- Remote session-ingress failure → `gracefulShutdownSync(1, 'other')` — fatal for managed deployments.
- Epoch mismatch (409) re-thrown to avoid race with graceful shutdown.

### 5.5 Shutdown pipeline

`gracefulShutdown.ts` (529 LOC) — single centralized 13-step exit pipeline + flat unordered `cleanupRegistry` `Set<() => Promise<void>>` (25 LOC, `cleanupRegistry.ts:23-25`, dispatched via `Promise.all`).

**Sync-first ordering** (load-bearing):
- `cleanupTerminalModes()` (`:59-136`, 11 internal steps) + `printResumeHint()` (`:144-184`) use `writeSync` **before any `await`**, so terminal sanity and resume hint are visible even under SIGKILL mid-cleanup.

**13-step pipeline** (`:391-523`):

```
 1. cleanupTerminalModes (sync, writeSync)
 2. printResumeHint (sync, writeSync, before alt-screen exit ordering matters)
 3. runCleanupFunctions (2000ms cap) — parallel Promise.all over registry
     ├─ Project.flush
     ├─ reAppendSessionMetadata  ← AFTER flush (64KB tail invariant)
     ├─ MCP stdio: SIGINT 100ms → SIGTERM 500ms → SIGKILL 600ms absolute
     ├─ Watchers / locks
     └─ Log forwarders
 4. executeSessionEndHooks (1500ms default, env-overridable)
 5. profileReport (BEFORE analytics shutdown — analytics cancels profile timers)
 6. tengu_cache_eviction_hint (event must reach pipeline)
 7. shutdown1PEventLogging + shutdownDatadog + OTEL shutdownTelemetry
 8. Analytics flush (500ms cap — can drop last ~500ms on slow networks)
 9. (final housekeeping)
10. forceExit(code) — always last
     └─ If process.exit() throws EIO (Bun on dead fd after SIGHUP/SSH-disconnect):
         process.kill(pid, 'SIGKILL')
```

**Global failsafe** — `max(5000, sessionEndTimeoutMs + 3500)` (gh-32712 fix). Without this term, a user's 10s hook budget was silently truncated to 5s.

**Exit codes**:
- `0` — normal
- `1` — ~80 validation/auth/trust-rejection sites
- `129` — SIGHUP + orphan detect
- `143` — SIGTERM

**`ExitReason` enum** (`src/entrypoints/sdk/coreSchemas.ts:747-754`): `clear` / `resume` / `logout` / `prompt_input_exit` / `other` / `bypass_permissions_disabled` — propagated to SessionEnd hooks.

**Ink handshake** (avoids DECRC cursor-reset double-fire):
- Register `Ink#unmount()` via `onExit` (signal-exit)
- `detachForShutdown()` **after** unmount — prevents deferred double-unmount clobber on tmux
- `DISABLE_MOUSE_TRACKING` **before** Ink unmount (round-trip to stop events)
- Exit alt-screen **before** `printResumeHint` (else hint lands on alt buffer and is lost on normal-screen restore)

**`gracefulShutdownSync`** (`:336-359`) — synchronous variant for non-resumable paths (remote-ingress fatal, etc.).

**43 registering files** across transports (MCP, LSP, bridge, tmux, shell tasks, swarm), persistence (session, config, history, activity), watchers/locks (settings, skills, keybindings, cron, CU, team memory, git-fs), output (OTEL, Perfetto, asciicast, NDJSON guard, terminal panel), and install (native, plugins, proxy). Registry is a `Set` — registration order **not** preserved — subsystems encode their own internal ordering (see MCP stdio escalation).

> **Cross-refs**: time bounds §7.6, abort tree §7.3, hooks §7.4, post-incident fingerprints §7.7. Background peers that register teardown callbacks are enumerated in §6.1.

## 6. Background & Alternative Modes (Cluster D)

Cluster D covers work happening *outside* the main REPL request cycle: parallel prefetches and housekeeping (6.1), alternative transports like MCP/bridge/remote-control (6.2), the daemon supervisor/worker topology (6.3), and the template-jobs subsystem (6.4). Two of these — daemon internals and templates — are feature-gated out of the extracted snapshot and are analyzable only through their contracts with neighboring code.

### 6.1 Background ring around REPL

Six tiers of concurrent activity run alongside the REPL without blocking it:

1. **Pre-render prefetches** — `setup.ts:95-380` fires UDS listeners, watchers, hooks capture, sinks, API-key resolution before first paint.
2. **Deferred prefetches** — `main.tsx:388-431` (`startDeferredPrefetches`) loads user context, system context, Statsig gates, MCP URLs, model caps *after* first render so paint is unblocked.
3. **Post-submit housekeeping** — `startBackgroundHousekeeping` (`backgroundHousekeeping.ts:31-94`) gated on `submitCount === 1` (REPL.tsx:3903-3907); runs MagicDocs, skill improvement, memory extraction, autoDream, plugin auto-update, very-slow-ops scheduler.
4. **Long-lived peers** — sinks (OTEL, Perfetto, analytics), `preventSleep` ref-counted caffeinate (`services/preventSleep.ts:36-58`), three chokidar watchers (`fileChangedWatcher`, `settingsChangeDetector`, `skillChangeDetector`).
5. **Per-turn transient tools** — Bash, subagents, tool subprocesses spawned per-turn; torn down at turn end.
6. **External peers** — MCP subprocesses, bridge/remote workers, cron tasks, UDS inbox listeners.

**Invariants across all tiers**:
- **`.unref()` discipline universal** — every timer and interval is `.unref()`'d so nothing holds the process alive during idle.
- **Housekeeping admission control** — reschedules if `getLastInteractionTime()` is within 60s, preventing slow I/O during active turns.
- **Single event loop** — "concurrent" means overlapping async, not threads. Heavy CPU must go to a subprocess.
- **Chokidar watchers are independent** — three separate instances don't de-dupe FS events; potential redundant work on shared paths (acknowledged).

**Cron integration**: fired tasks enqueue at `'later'` priority via `useScheduledTasks` (`useScheduledTasks.ts:40`); REPL's `useCommandQueue` drains them between turns. Permanent tasks are partitioned between daemon and non-daemon processes via a `permanent` flag in `scheduled_tasks.json` — daemon claims the permanent set when running.

**Transport exclusivity** — bridge, remote, direct, and SSH transports are mutually-exclusive: `activeRemote = sshRemote ?? directConnect ?? remoteSession` (REPL.tsx:1421-1422). Only one receives messages per session.

### 6.2 MCP / Bridge / Remote transports

Three REPL-hosted alternative transports share the cleanup registry and re-use the same request-cycle core (`query.ts`) but differ in how they receive user input.

**MCP** (`src/services/mcp/client.ts`, 3348 LOC):
- Stdio children (`mcp__server__tool` in tool pool), remote HTTP/SSE/WS variants.
- `claude.ai` connector has a bounded wait (`CLAUDE_AI_MCP_TIMEOUT_MS`) to avoid blocking tool-pool assembly if the connector is slow.
- Shutdown: `SIGINT` 100ms → `SIGTERM` 500ms → `SIGKILL` 600ms absolute failsafe; 100+ LOC inner state machine (`mcp/client.ts:1429-1562`) with orphan-then-let-OS-reap fallback.

**Bridge / Remote-Control** (`src/bridge/*`, ~12.6k LOC across 31 files):
- Three entrypoint shapes converge on one loop:
  - Standalone: `claude remote-control` → `bridgeMain.runBridgeLoop`
  - REPL-hosted: `/remote-control` command → `replBridge.initBridgeCore`
  - Daemon-worker: `runBridgeHeadless` → same loop via `HeadlessBridgeOpts`
- **v1 vs v2 protocol split**: v1 is env-based (register/poll/ack/heartbeat against Environments API, WS or SSE transport); v2 is env-less SSE+CCR only, uses `/bridge` endpoint returning `worker_jwt` + `worker_epoch`. v2 is REPL-only today — daemon, standalone, and print still take v1.
- **Init-bypass** — all bridge entrypoints skip `init.ts` and re-do only `enableConfigs` / `initSinks` / CWD / trust. `bridgeMain` must call `process.exit(0)` explicitly (`bridgeMain.ts:2767`) because nothing else cleans up.
- **Auth**: `getClaudeAIOAuthTokens` with proactive refresh; REPL adds cross-process dead-token backoff via global config keys `bridgeOauthDeadExpiresAt` / `bridgeOauthDeadFailCount` to prevent retry amplification across parallel REPLs.
- **CCR session-id retagging** — `session_*` ↔ `cse_*` id translation happens at ~30 call sites in v2; a shim abstracts this so the rest of the codebase sees consistent ids.

### 6.3 Daemon architecture (contract-only)

Daemon source is excluded from the extracted tree. Both `src/daemon/main.ts` and `src/daemon/workerRegistry.ts` are referenced by `cli.tsx:100-106` and `cli.tsx:164-180` but gated out by `feature('DAEMON')` at build. Only the *contract* with the daemon is analyzable.

**Topology** — `claude daemon` supervises workers spawned as `claude --daemon-worker <kind>`. Workers skip `enableConfigs()` / `initSinks()` at dispatch for cold-start latency and opt in themselves; supervisor runs both inline.

**Worker kinds observed**:
- `remoteControl` (bridge, visible via `runBridgeHeadless` — only analyzable worker body)
- `cron` (inferred from `cronScheduler.ts:123-124` — handles `permanent: true` tasks)
- `assistant` (inferred from `main.tsx:1052-1056` — Kairos / Agent-SDK, hidden `--assistant` flag skips Kairos recheck)

**Shared auth via IPC** — supervisor owns the `AuthManager`; workers call `opts.getAccessToken()` / `opts.onAuth401()` (defined by `HeadlessBridgeOpts`, `bridgeMain.ts:2785-2797`) rather than reading OAuth state directly. Avoids lock contention and duplicated refreshes.

**Error taxonomy (load-bearing)**:
- `BridgeHeadlessPermanentError` → `EXIT_CODE_PERMANENT` → supervisor "parks" the worker (no respawn)
- Plain `Error` → transient → respawn with backoff
- Classification is split between preflight checks and a runtime `fatalExit` flag

**v1/v2 token-refresh split** (`bridgeMain.ts:279-313`) traces to CC-1263 — v2 daemon sessions refresh tokens differently from v1 to avoid a specific race. Supervisor-restart retry is upstream proxy's assumed recovery model (`upstreamproxy.ts:13,139`).

**Cross-process coordination**:
- `daemon.json` — supervisor config (schema not extracted)
- `~/.claude/sessions/<pid>.json` — PID registry tagged `daemon` / `daemon-worker` for `claude ps`
- `scheduled_tasks.json` — partitioned by `permanent` flag
- `CLAUDE_CODE_MESSAGING_SOCKET` — UDS inbox, possible supervisor↔worker IPC channel

**Known gap** — WSL pruning of stale daemon-worker PID files (`concurrentSessions.ts:194-201`) — accumulating entries possible cross-platform.

### 6.4 Templates (contract-only)

`cli.tsx:211-222` dispatches `claude new | list | reply` to `src/cli/handlers/templateJobs.js`, but that file is not present in the extracted tree. `feature('TEMPLATES')` gates it out of external builds.

Only the dispatch contract is analyzable:
- Three subcommands (`new`, `list`, `reply`) handled by the template jobs module
- Fast-path dispatch before full CLI init (same pattern as bridge/daemon)
- No other entry points touch templates — the subsystem is self-contained behind its handler file

Full analysis requires an internal build with `feature('TEMPLATES')` enabled.

> **Cross-ref**: cleanup and abort discipline for all Cluster D components is documented in §7.3 (abort tree) and §7.6 (time-bounded async). Trust checks after init-bypass are in §7.1.

## 7. Cross-cutting Concerns

Seven disciplines thread through ≥3 of the four clusters. Each is lifted here rather than duplicated per cluster; cluster sections reference back by callout.

### 7.1 Trust boundary

**Threads through**: A (establishes), B (gates prompt content + env var application), C (forceLogin invariants on resume), D (bridge policy check before dispatch).

A single gate — `showSetupScreens` — stands between setup and any code that could exfiltrate data or act on untrusted input. Passing the gate unlocks, in order:
- `applyConfigEnvironmentVariables` (user `env` block applied to `process.env`)
- `resetGrowthBook` + `initializeGrowthBook` (authenticated flag evaluation)
- OAuth / API-key helper invocation
- Telemetry init (`initSinks` second call path)
- `updateGithubRepoPathMapping`
- Managed-env apply from MDM policy

Trust state is a session-scoped boolean plus a per-directory disk record; parent-walk propagates trust into subdirectories; only `true` is memoized (re-check after untrust toggle). The home directory **never** persists trust — it must be re-granted each session. Enterprise `forceLoginOrgUUID` uses an uncached profile read to avoid reading stale tokens across restarts. Bridge entrypoints (`bridgeMain.ts`, `initReplBridge.ts`) re-check trust after init-bypass because they skip `init.ts`.

### 7.2 Prompt cache discipline

**Threads through**: A (built-in tool prefix ordering, beta-header latches in Settings), B (system-prompt boundary marker, byte-exact text preservation through parsing round-trips), C (`/clear` and `/compact` clear the latches).

One cache boundary marker in the system prompt is the load-bearing invariant. If the boundary drifts by a single byte, every request goes uncached — cost and latency cliff. Disciplines:
- **Tool prefix ordering**: built-in tools precede MCP tools in `assembleToolPool`; MCP tools ordered by server then name (`mcp__server__tool`).
- **Byte-exact text preservation**: parsing never re-normalizes whitespace or re-serializes content blocks; the accumulator writes text exactly as received.
- **Sticky beta-header latches**: beta headers, once set for a session, remain set until `/clear` or `/compact` explicitly clear them. This prevents header-churn cache misses across turns.
- **Boundary clear on user-initiated reset only**: `/clear` resets session + latches; `/compact` writes a `compact_boundary` entry and resets latches. Automatic compaction (from 5-way stack) does not.

### 7.3 Abort / cancellation tree

**Threads through**: A (teardown on setup failure), B (tool/API cancellation mid-turn), C (shutdown entry point), D (bridge/daemon worker abort, session handoff).

Three non-overlapping layers compose:
1. **OS signals** — `SIGINT`/`SIGTERM`/`SIGHUP` + `uncaughtException`/`unhandledRejection` → `void gracefulShutdown()` (`gracefulShutdown.ts:237-334`).
2. **In-process `AbortController` tree** — root per REPL turn; children via `createChildAbortController` with `WeakRef` to avoid memory pinning (`abortController.ts:16-22`, `:68-99`). `maxListeners = 50`.
3. **API retry loop** — `withRetry` checks `signal?.aborted` at every loop header; `sleep(delayMs, signal, { abortError })` aborts mid-sleep (`withRetry.ts:170-517`).

**Semantic abort reasons** (dispatched via `signal.reason`): `'user-cancel'` (Esc / Ctrl+C), `'background'` (Ctrl+B handoff), `'interrupt'` ('now'-priority injected command). Consumers dispatch differentially — abandon vs handoff vs requeue.

**`isAbortError` triple-check** (`errors.ts:27-33`): `AbortError | APIUserAbortError | name === 'AbortError'` — because SDK class names are minified in production builds and `instanceof` alone is unreliable.

### 7.4 Hooks extension surface

**Threads through**: A (`SessionStart`, `PreToolUse` registered at mount), B (`PreToolUse`/`PostToolUse` per tool, `Stop` hooks at turn end), C (`SessionEnd`, `PermissionDenied`, `TaskCompleted`, `TeammateIdle`), D (file-watcher, settings/skill change).

15+ hook event types; hooks are first-class extension points that can:
- **Rewrite inputs** before tool execution (`PreToolUse`).
- **Override permissions** (`PreToolUse` return value).
- **Prevent continuation** (`Stop` hook return blocks the follow-up turn).
- **Retry on denial** (`PermissionDenied` can prompt re-attempt).

**MCP tool post-hook ordering differs**: for MCP tools, `PostToolUse` hooks run *before* result merge, so they can rewrite output — the built-in tool path runs hooks after merge. Intentional asymmetry because MCP tool results are opaque JSON.

`executeSessionEndHooks` budget (`CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`, default 1500ms) is load-bearing: the failsafe budget downstream is `max(5000, hookBudget + 3500)`, and gh-32712 silently truncated user hook budgets before that fix.

### 7.5 Subagent / teammate scoping

**Threads through**: B (`toolUseContext.agentId` gates tool filtering, async agent tool allowlist), C (per-agent persistence files, `TaskCompleted` hooks on subagent exit), D (teammate via `InProcessTeammateTask`, swarm backends).

Subagents run with narrower capabilities than the parent session:
- **`shouldAvoidPermissionPrompts = true`** forced for all subagents — no interactive permission UI.
- **Inherited parent mode + cwd** — subagent respects parent's plan-mode / accept-edits-mode posture.
- **Async agents exclude MCP tools** — the async-agent tool allowlist is narrower than the sync subagent allowlist.
- **`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`** is the teammate-specific allowlist (e.g., `SendMessage`, `TaskUpdate`).

Persistence is agent-scoped: each subagent writes its own JSONL sidechain; writes bypass the main dedup set (the parent's chain filter doesn't apply).

### 7.6 Time-bounded async / `.unref()` discipline

**Threads through**: B (idle watchdog 90s, stall 30s, stream timeout), C (registry 2s, SessionEnd 1.5s, analytics 500ms, failsafe `max(5s, hook+3.5s)`), D (MCP `SIGINT→SIGTERM→SIGKILL` escalation 100→500→600ms, housekeeping admission control).

**Two rules, universal**:
1. Every async phase has a cap — no unbounded `await`.
2. Every background timer is `.unref()`'d — no timer holds the process alive.

Specific caps by subsystem (selection):
- **Stream**: idle watchdog 90s, stall 30s, hard timeout per-model.
- **Cleanup registry**: 2000ms (global `Promise.all`).
- **SessionEnd hooks**: 1500ms default, env-overridable.
- **MCP stdio escalation**: `SIGINT` 100ms → `SIGTERM` 500ms → `SIGKILL` 600ms absolute failsafe (`mcp/client.ts:1429-1562`).
- **Analytics flush**: 500ms (acknowledged — last ~500ms of events can drop on slow networks).
- **Global failsafe**: `max(5000, hookBudget + 3500)` (gh-32712 fix).

**Sync-first for user-visible state**: `cleanupTerminalModes` + `printResumeHint` use `writeSync` *before* any `await`, so terminal sanity and resume hint are visible even under `SIGKILL` mid-cleanup.

### 7.7 Post-incident invariants

The single strongest signal that this codebase is production-hardened. Every scar below is a comment-anchored invariant tracing back to a specific incident. Breaking any of them re-opens the incident.

| Incident | Invariant | Location |
|---|---|---|
| **CC-34** | Session ID + projectDir atomicity (`switchSession` atomic pair) | `sessionStorage.ts` |
| **gh-32712** | Failsafe budget `max(5000, hookBudget + 3500)` — prevents silent hook truncation | `gracefulShutdown.ts` |
| **inc-3694, inc-4029, inc-4258** | Streaming / parsing fallback (tombstone + discard + rebuild) | `query.ts`, `claude.ts` |
| **inc-3930** | Tombstone OOM — 50MB slow-path cap on `removeMessageByUuid` | `sessionStorage.ts:871-951` |
| **#21056** | Permission bubble-up for `ExitPlanMode` | permission UI |
| **#14373, #23537** | Progress-in-chain (preservedSegment ordering) | `sessionStorage.ts:1839-1956` |
| **#22453** | Resume truncation (64KB tail window) | `sessionStorage.ts:721-839` |
| **ANT-344** | Persistent-retry keep-alive dedicated channel (TODO) | `withRetry.ts:477-506` |
| **adamr-20260320** | Fork-session sessionId cross-contamination fix | `sessionRestore.ts` |
| **gh-30217** | `transcript_path` cwd mismatch (worktree chdir timing) | `setup.ts:176-285` |
| **CC-1263** | v2 daemon session token refresh | `bridgeMain.ts:279-313` |
| **Bun sigaction pin** | `onExit(() => {})` no-op subscriber prevents signal-exit v4 `removeListener` from resetting kernel sigactions | `gracefulShutdown.ts:254` |
| **Bun `AbortSignal.timeout` leak** | Plain `setTimeout` instead (2.4KB native-memory leak per timeout) | `combinedAbortSignal.ts:15-47` |
| **macOS TTY revoke** | 30s `.unref()`'d orphan detect — `stdout.writable && stdin.readable` poll | `gracefulShutdown.ts:281-296` |
| **EIO on dead TTY** | `process.exit` EIO → `process.kill(pid, 'SIGKILL')` fallback | `gracefulShutdown.ts:193-232` |

The density of these references — at least 15 distinct post-incident fixes across the 17 subsystems examined — is the defining characteristic of the codebase. Treat any code comment referencing an incident or issue number as load-bearing.

## 8. Appendices (Reference Material)

Flat reference tables extracted from Wave 1 reports. Tables only; narrative belongs to §§3–7.

### Appendix A — `withRetry` state machine

Source: `src/services/api/withRetry.ts` (822 LOC).

| State / Path               | Trigger                                               | Action                                                   | Location          |
|----------------------------|-------------------------------------------------------|----------------------------------------------------------|-------------------|
| Abort check                | `signal?.aborted` at loop top                         | Throw `AbortError`                                       | `:170-179`        |
| Retryable error            | 5xx, 408, 429, connection-timeout                     | Backoff + jitter, respect `Retry-After`                  | `:190-265`        |
| Non-retryable error        | 4xx (non-408/429), refusal, context-window            | Rethrow                                                  | `:190-265`        |
| Foreground 529             | Source in `FOREGROUND_529_RETRY_SOURCES`              | Retry up to `MAX_529_RETRIES=3`                          | `:62-82`          |
| Background 529             | Any (dropped — cascade amplification)                 | Throw immediately                                        | `:62-82`          |
| Fallback after 529         | `MAX_529_RETRIES` exhausted                           | Throw `FallbackTriggeredError`                           | `:326-365`        |
| Fast-mode cooldown         | Fast mode triggered by specific errors                | `triggerFastModeCooldown` → `utils/fastMode.ts`          | `:267-314`        |
| Opus → fallback chain      | Model-under-load on primary                           | Escalate through model chain                             | `:326-365`        |
| Persistent mode            | `CLAUDE_CODE_UNATTENDED_RETRY=1`                      | 30s yields with heartbeat (ANT-344 keep-alive TODO)      | `:477-506`        |
| Context-overflow self-heal | 413 / `model_context_window_exceeded`                 | In-loop compaction                                       | `:550-595`        |
| Bedrock auth recovery      | Bedrock 401/403                                       | Clear creds cache + retry                                | `:631-694`        |
| Vertex auth recovery       | Vertex 401/403                                        | Clear creds cache + retry                                | `:631-694`        |
| OAuth auth recovery        | Anthropic 401                                         | Refresh (deduped by token value, cross-process lock)     | `auth.ts:1360-1562`|
| API-key auth recovery      | Anthropic 401 with API key                            | Provider-specific clear                                  | `:631-694`        |
| Sleep mid-retry            | Backoff window                                        | `sleep(delayMs, signal, { abortError })` — aborts mid    | `:190-265`        |

### Appendix B — `Continue` × `Terminal` reason matrix

Source: `src/query/transitions.ts` (file not in snapshot; enumerated from `query.ts` usages).

**Continue reasons** (7) — loop iterates again:

| Reason                           | Trigger                                                | Notes                                         |
|----------------------------------|--------------------------------------------------------|-----------------------------------------------|
| `collapse_drain_retry`           | Context-collapse drain needed                          | One-shot — checked via `state.transition?.reason` |
| `reactive_compact_retry`         | 413 → reactive compaction                              | `hasAttemptedReactiveCompact` preserved       |
| `max_output_tokens_escalate`     | `max_tokens` stop, budget available to escalate        | Reuses max-output-tokens recovery             |
| `max_output_tokens_recovery`     | `max_tokens` / `model_context_window_exceeded`         | Recovery path                                 |
| `stop_hook_blocking`             | Stop hook returned blocking errors                     | Retry with injected errors                    |
| `token_budget_continuation`      | `TOKEN_BUDGET` gate, budget nudge injected             | Subagents skipped; diminishing-returns guard  |
| `next_turn`                      | `needsFollowUp === true`, normal continue              | Most common path                              |

**Terminal reasons** (12) — loop exits with this code:

| Reason                           | Trigger                                                | Exit behavior                                 |
|----------------------------------|--------------------------------------------------------|-----------------------------------------------|
| `completed`                      | `needsFollowUp === false`, no tool_use                 | Normal success                                |
| `aborted_streaming`              | Abort during stream                                    | Reason on `signal.reason`                     |
| `aborted_tools`                  | Abort during tool execution                            | Tool results may be partial                   |
| `blocking_limit`                 | Hard message-limit gate after compaction               | Post-compaction limit trip                    |
| `image_error`                    | Image processing failure                               | Held error replayed                           |
| `model_error`                    | `FallbackTriggeredError` after 529s                    | Bubbles to outer                              |
| `prompt_too_long`                | Unrecoverable 413                                      | Reactive compact exhausted                    |
| `stop_hook_prevented`            | Stop hook `preventContinuation: true`                  | User intent                                   |
| `hook_stopped`                   | SessionEnd or external hook stop                       | Respects hook authority                       |
| `max_turns`                      | `maxTurns` reached (`query.ts:1704-1712`)              | Configurable                                  |
| `refusal`                        | AUP refusal                                            | Synthesized message                           |
| `other`                          | Catch-all / unknown                                    | Rare                                          |

Invariants:
- `stop_reason === 'tool_use'` **not** read — content presence (`needsFollowUp`) authoritative.
- `collapse_drain_retry` is one-shot.
- `hasAttemptedReactiveCompact` preserved across `stop_hook_blocking` retries (death-spiral guard at `query.ts:1292-1297`).

### Appendix C — Error taxonomy (selection)

Source: `src/services/api/errors.ts` (~30 error classes); `src/utils/errors.ts`.

| Error                          | Source            | Retryable | Notes                                                      |
|--------------------------------|-------------------|-----------|------------------------------------------------------------|
| `APIError`                     | SDK               | Depends   | Base class                                                 |
| `APIConnectionError`           | SDK               | Yes       | Connection layer                                           |
| `APIConnectionTimeoutError`    | SDK               | Yes       | Rethrown from abort-vs-timeout (claude.ts:2434-2462)       |
| `APIUserAbortError`            | SDK               | No        | Maps through `isAbortError` (minified class name)          |
| `AbortError`                   | Built-in / custom | No        | Triple-checked (`errors.ts:27-33`)                         |
| `AuthenticationError` (401)    | SDK               | After auth refresh | Deduped refresh; filesystem lock                   |
| `PermissionDeniedError` (403)  | SDK               | Depends   | `insufficient_scope` → step-up PKCE                        |
| `RateLimitError` (429)         | SDK               | Yes       | `Retry-After` respected                                    |
| `InternalServerError` (5xx)    | SDK               | Yes       | Backoff + jitter                                           |
| `OverloadedError` (529)        | SDK               | Foreground only | `MAX_529_RETRIES=3` → `FallbackTriggeredError`       |
| `FallbackTriggeredError`       | withRetry         | No        | Bubbles to query loop; reactive compact / Opus fallback    |
| `BadRequestError` (400)        | SDK               | No        | Unless 413-flavored                                        |
| 413 / prompt-too-long          | SDK               | Via compact | Reactive-compact recovery path                          |
| `model_context_window_exceeded`| SDK               | Via compact | Reuses max-output-tokens recovery                        |
| `refusal`                      | Response          | No        | `getErrorMessageIfRefusal` synthesizes AUP message         |
| `ConfigParseError`             | init.ts           | Interactive recovery | invalid-config dialog (`init.ts:215-236`)         |
| `BridgeHeadlessPermanentError` | bridge            | No        | → `EXIT_CODE_PERMANENT`; supervisor parks worker           |

### Appendix D — Exit codes + `ExitReason` enum

**Exit codes**:

| Code | Trigger                                                  | Notes                                      |
|------|----------------------------------------------------------|--------------------------------------------|
| 0    | Normal exit                                              | `/exit`, Ctrl+D, SIGINT (non-print)        |
| 1    | Validation / auth / trust-rejection (~80 sites)          | TrustDialog decline, MDM policy reject     |
| 129  | SIGHUP + orphan detect                                   | TTY revocation (macOS)                     |
| 143  | SIGTERM                                                  | External kill                              |
| `EXIT_CODE_PERMANENT` | `BridgeHeadlessPermanentError` | Supervisor parks worker (daemon)     |

**`ExitReason`** enum (`src/entrypoints/sdk/coreSchemas.ts:747-754`):

| Value                          | Propagated to                          |
|--------------------------------|----------------------------------------|
| `clear`                        | SessionEnd hooks                       |
| `resume`                       | SessionEnd hooks (transition marker)   |
| `logout`                       | SessionEnd hooks                       |
| `prompt_input_exit`            | SessionEnd hooks                       |
| `other`                        | SessionEnd hooks (default)             |
| `bypass_permissions_disabled`  | SessionEnd hooks (policy-driven)       |

### Appendix E — `SessionKind` + PID registry schema

Source: `src/utils/concurrentSessions.ts:18,27-37`.

**`SessionKind`** values:

| Kind              | Entrypoint                              | Ink tree | Notes                                        |
|-------------------|-----------------------------------------|----------|----------------------------------------------|
| `interactive`     | `cli.tsx` → REPL                        | yes      | Full init; mounts Ink tree                   |
| `bg`              | `cli.tsx --bg`                          | no       | Headless background session                  |
| `daemon`          | `cli.tsx --daemon` (supervisor)         | no       | Feature-gated (`DAEMON`)                     |
| `daemon-worker`   | `cli.tsx --daemon-worker <kind>`        | no       | Init-bypass; IPC via supervisor              |

Override via `CLAUDE_CODE_SESSION_KIND` env.

**PID registry** — `~/.claude/sessions/<pid>.json`:

| Field                | Type         | Description                                   |
|----------------------|--------------|-----------------------------------------------|
| `pid`                | number       | Process id                                    |
| `sessionKind`        | SessionKind  | One of the four above                         |
| `sessionId`          | string       | Active session id                             |
| `projectDir`         | string       | Sanitized cwd path                            |
| `startedAt`          | ISO8601      | Registration time                             |
| `bridgeId`           | string?      | Set via `updateSessionBridgeId` for remote    |
| `workerKind`         | string?      | For daemon-workers: `remoteControl`/`cron`/...|
| `title`              | string?      | Session customTitle or derived                |

`claude ps` reads this directory; stale entries pruned on startup (WSL gap noted — `concurrentSessions.ts:194-201`).

**Related registries**:
- `daemon.json` — supervisor config (schema not extracted)
- `scheduled_tasks.json` — partitioned by `permanent` flag (daemon vs non-daemon)
- `CLAUDE_CODE_MESSAGING_SOCKET` — UDS inbox path

### Appendix F — JSONL `Entry` types

Source: `src/utils/sessionStorage.ts:1128-1265` (`Project.appendEntry`).

Discriminated by `type` field. 18+ types (selection — order not fixed in file; logical grouping):

| `type`                         | Payload                              | Purpose                                            |
|--------------------------------|--------------------------------------|----------------------------------------------------|
| `message`                      | Full `Message`                       | User / assistant message with `parentUuid`         |
| `summary`                      | Summary text                         | Session-level summary metadata                     |
| `compact_boundary`             | Summary + usage                      | Autocompact anchor (parentUuid=null, logicalParent)|
| `preservedSegment`             | UUID range                           | Relinks for in-memory splice                       |
| `snipMetadata`                 | `{ removedUuids }`                   | Path-compression of user-snipped ranges            |
| `marble-origami-commit`        | Ordered log entry                    | Context-collapse commit                            |
| `marble-origami-snapshot`      | Last-wins snapshot                   | Context-collapse snapshot                          |
| `customTitle`                  | String                               | User-renamed title (re-appended; precedence)       |
| `ai-title`                     | String                               | Haiku-generated title (**never** re-appended)      |
| `tag`                          | String                               | User tag (re-appended to 64KB window)              |
| `gitBranch`                    | String                               | Current branch at turn                             |
| `prInfo`                       | PR metadata                          | GitHub PR linkage                                  |
| `agent`                        | Agent metadata                       | Subagent session info                              |
| `worktree-state`               | `undefined` / `null` / object        | Tri-state worktree tracking                        |
| `tengu_exit`                   | Exit reason + code                   | Replayed next session (setup Phase L)              |
| `toolResultStorage-freeze`     | Freeze marker                        | FROZEN classification on resume                    |
| `sessionStamp`                 | Session id + projectDir              | CC-34 atomicity witness                            |
| `recordContentReplacement`     | Pre-fork seed                        | Fork-session content relinks                       |

**Invariants across all types**:
- `parentUuid` must be the first JSON key — `walkChainBeforeParse` relies on `{"parentUuid":` prefix for dead-fork elimination.
- `ai-title` intentionally not re-appended (customTitle precedence).
- Compact boundary: `parentUuid = null`, `logicalParentUuid = originalParent`.
- Agent sidechain writes bypass main dedup set.
- Files `0o600`, directories `0o700`.

## 9. Recommendations for Follow-up

Wave 1 covered the four stated goals (launch sequence, REPL loop, background processes, alternative modes) with sufficient depth for architectural understanding. This section collects what would be needed to close remaining gaps — organized by whether the gap is a *source availability* problem, a *depth* problem, or a *new-question* problem.

### 9.1 Re-extract with additional feature gates

The following subsystems are gated out of the extracted snapshot. Only call-site contracts are analyzable today. Re-extraction with the matching `feature(...)` flags enabled would unblock full analysis.

| Gate                 | Missing source                                              | What it unlocks                                                           |
|----------------------|-------------------------------------------------------------|---------------------------------------------------------------------------|
| `feature('DAEMON')`  | `src/daemon/main.ts`, `src/daemon/workerRegistry.ts`        | Supervisor lifecycle, backoff policy, worker-kind dispatch, shutdown flow |
| `feature('TEMPLATES')` | `src/cli/handlers/templateJobs.js`                        | Template fleet view (`claude new/list/reply` implementations)             |
| `feature('KAIROS')`  | assistant-worker module referenced by `main.tsx:1052-1056`  | Agent-SDK integration, `--assistant` flag internals                       |
| `"external" === 'ant'` | Event-loop stall detector, SDK heap-dump monitor, ANT-344 persistent-retry channel | Internal observability; ant-only diagnostics              |

### 9.2 Missing files (extraction artifacts, not gated)

These are referenced but absent from the snapshot. They are not feature-gated — likely extraction-script omissions:

- **`src/query/transitions.ts`** — imported at `query.ts:104`; Continue/Terminal type definitions. Behavior is enumerated in Appendix B from usages, but the source would confirm completeness.
- **MCP connection pool internals** — `services/mcp/client.ts` is present (3348 LOC) but connection-pool helpers referenced from it may live in adjacent files not extracted.

### 9.3 Deep-dive opportunities (architectural depth)

High-value follow-ups where Wave 1 identified rich complexity worth dedicated treatment:

1. **Compaction stack** (5 interacting mechanisms)
   - Why: Snip, microcompact, collapse, autocompact, reactive — each has ordering invariants tied to specific incidents. The 5-way interaction warrants its own reference document with state diagram.
   - Sources: `query.ts:365-548`, `compact.ts`, `reactiveCompact.ts`, `contextCollapse.ts`.

2. **`StreamingToolExecutor` internals** (TrackedTool, sibling aborts, discard/rebuild)
   - Why: Mid-stream tool dispatch with fallback/rebuild is intricate; fallback interacts with executor state in non-obvious ways (`query.ts:712-740` ↔ `StreamingToolExecutor.ts:40-205`).
   - Followup: Sequence diagram of the streaming-fallback dance.

3. **Permission + hook state machine** (`toolExecution.ts:921-1103`)
   - Why: Quadruple state (`hookPermissionResult`, `hookUpdatedInput`, `shouldPreventContinuation`, `stopReason`) is load-bearing but scattered.
   - Followup: State-machine diagram + MCP vs non-MCP ordering reference.

4. **`withRetry.ts` full state machine** (822 LOC)
   - Why: Four auth-refresh paths, fast-mode dual path, persistent-mode heartbeat, Opus fallback chain, context-overflow self-heal, provider-specific (Bedrock, Vertex) recovery.
   - Followup: Condensed reference table (partial draft in Appendix A).

5. **Bridge session-ingress protocol** (v1 WS vs v2 SSE+CCR)
   - Why: Wire format, sequence numbering, reconnection logic, CCR `session_*`↔`cse_*` retagging at ~30 call sites.
   - Sources: `replBridgeTransport.ts`, `bridgeMessaging.ts`, `ccrClient.ts`.

6. **Token-refresh scheduler** (`jwtUtils.ts`)
   - Why: Three refresh flows with subtle laptop-wake races; cross-process dead-token backoff.
   - Followup: Race-condition catalog + recovery matrix.

7. **Session-persistence edge cases** (sessionStorage.ts, 5105 LOC)
   - Why: Already "most carefully engineered subsystem examined" — but fileHistory, toolResultStorage FROZEN classification, `deserializeMessagesWithInterruptDetection` legacy migrations would round out coverage.
   - Followup: Resume picker progressive enrichment (`ResumeConversation.tsx`).

8. **OAuth refresh + 401 machinery**
   - Why: Token refresh, downstream auth cascades, provider-specific recovery paths. Separate concern from trust boundary architecture.
   - Sources: `utils/auth.ts`, `services/api/withRetry.ts:631-694`.

### 9.4 Reference-material deep-dives (flat catalogs)

Worth assembling as standalone reference documents:

- **Error taxonomy** — ~30 error codes in `errors.ts` with handling map.
- **Hook event types** — 15+ events, their signatures, and registration surfaces.
- **Exit codes + `ExitReason` enum** — full matrix with trigger conditions.
- **`SessionKind` registry + PID file schema** — `concurrentSessions.ts` formats.
- **JSONL `Entry` types** — 18+ discriminated types in `Project.appendEntry`.
- **Slash-command registry** — pre-loop vs in-loop split, skill vs command dispatch.

Appendix sections in this report cover some of these (A, B, C, D, E, F); others remain.

### 9.5 Tool-specific investigations

Each is valuable when building or debugging the specific tool:

- **Bash tool + command classifier** — subprocess tree kill, escape handling.
- **MCP tool generation/router** — dynamic tool registration, prefix rules.
- **AgentTool / Task subsystem** — subagent spawn, budget accounting, `TaskCompleted`.
- **ShellCommand / computerUse cancellation** — subprocess-tree semantics.

### 9.6 Operational / testing investigations

Gaps in the stated-goal coverage that surfaced during analysis:

- **Double-call idempotency** for `gracefulShutdown` — test matrix partially implied.
- **Failsafe-fired path coverage** — when `max(5000, hookBudget + 3500)` hits.
- **EIO → SIGKILL fallback** — dead-TTY reproduction harness.
- **Print-mode SIGINT override** — ensures print-mode doesn't fall through to global handler.
- **Resume hint vs flush failure** — user told to `--resume` but file may be inconsistent (intentional trade-off; worth documenting).
- **Chokidar watcher overlap** — three independent instances, no FS-event de-dup; measure redundancy.
- **WSL PID-file pruning** — stale daemon-worker entries can accumulate (`concurrentSessions.ts:194-201`).

### 9.7 Longitudinal investigations

Questions that require data beyond the source:

- **`tengu_streaming_stall` threshold tuning** — telemetry data would inform the 30s/90s constants.
- **Cache-ttl decision outcomes** — 5m vs 1h selection effectiveness per model.
- **Analytics 500ms budget impact** — how often last-events drop on slow networks.
- **Persistent-retry keep-alive (ANT-344)** — dedicated channel design awaiting implementation.

### 9.8 Priority summary

| Priority | Category | Effort | Blocker for?          |
|----------|----------|--------|-----------------------|
| High     | 9.1 Re-extract with `DAEMON` + `TEMPLATES` gates | Low (build config) | Full Cluster D coverage |
| High     | 9.3.1 Compaction stack deep-dive | Medium | Incident response on compact paths |
| Medium   | 9.3.2 StreamingToolExecutor diagram | Medium | Streaming-fallback debugging |
| Medium   | 9.3.3 Permission+hook state machine | Medium | Tool security invariants |
| Medium   | 9.2 Locate `transitions.ts` | Low | Appendix B completeness |
| Low      | 9.4 Reference catalogs | Low each | Onboarding / reference |
| Low      | 9.5 Tool-specific | Varies | Tool authorship |
| Low      | 9.6 Operational/testing | Medium | Reliability regression guards |


