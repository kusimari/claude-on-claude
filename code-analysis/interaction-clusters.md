# Synthesis Plan — Interaction Clusters and Grouping for Final Report

**Inputs**: `summaries/phase-1-summary.md` through `summaries/phase-4-summary.md`, `wave-1-review.md`, 17 Wave 1 deep-dive reports.

**Goal**: Organize 17 subsystems into a coherent narrative for `codebase-report.md` by identifying (a) subsystems documentable independently, (b) interaction clusters with tight coupling, (c) cross-cutting concerns that thread through multiple clusters.

---

## 1. Methodology

A subsystem belongs to a cluster if it participates in a shared invariant, data flow, or lifecycle choreography with at least two other subsystems. A concern is cross-cutting if its discipline must be maintained across ≥3 clusters. Subsystems outside any cluster are "independent" and documented standalone.

Interactions observed from the summaries fall into four kinds:
- **Data flow** — one produces values the other consumes (streaming events, messages, tool results).
- **Lifecycle** — ordered setup/teardown handshakes (bootstrap, shutdown).
- **Shared invariants** — byte-stable ordering, cache discipline, signal semantics.
- **Extension contracts** — plug-in points (hooks, MCP, tools, subagents).

---

## 2. Cluster Map (high level)

```
                  ┌────────────────────────────────┐
                  │  Cluster A: Bootstrap          │
                  │  Setup · Trust · Tools · MCP · │
                  │  State/Config                  │
                  └────────────┬───────────────────┘
                               │ (produces: trusted env, tool pool,
                               │  MCP pool, session state)
                               ▼
┌────────────────────┐   ┌────────────────────────────────┐   ┌──────────────────────┐
│ Cluster D:         │   │  Cluster B: Request Cycle      │   │ Cluster C: Lifecycle │
│ Background &       │◀──▶│  REPL · Query · Prompt · API · │◀──▶│ Interrupts ·         │
│ Extensibility      │   │  Parsing · Inner Agent Loop    │   │ Persistence ·        │
│ Concurrent procs · │   │                                │   │ Cleanup              │
│ Daemon · Bridge ·  │   └────────────────────────────────┘   └──────────────────────┘
│ Templates          │
└────────────────────┘

                Cross-cutting concerns thread through all 4:
   [Trust boundary] [Prompt cache discipline] [Abort tree]
   [Hooks system]   [Subagent scoping]        [Time-bounded async]
```

---

## 3. Cluster A — Bootstrap (Reports 01–05)

**Members**: 01 setup, 02 trust/auth, 03 tool-system-init, 04 MCP-bootstrap, 05 state/config/settings.

**Why they cluster**: every member contributes to the pre-REPL readiness contract. They share a strict ordering, a shared trust boundary, and produce the inputs the Request Cycle consumes.

**Shared lifecycle ordering** (from summaries):
```
enableConfigs → MDM awaited → first settings access → setup() phases A-L
  → showSetupScreens (trust gate) → env-var apply → telemetry init
  → tool pool assembled → MCP pool kicked off → first render
```

**Critical interactions**:
- **Trust → env/GrowthBook/OAuth/GitHub mapping**: `showSetupScreens` is the single gate that unlocks `applyConfigEnvironmentVariables`, `resetGrowthBook+initializeGrowthBook`, API key helper, and `updateGithubRepoPathMapping`. Reports 02 + 05 both document the invariant.
- **Settings → tool pool**: `permissions.allow/deny/ask` feed `assembleToolPool`; `--allowedTools`/`additionalDirectories` flow through `toolPool.ts`.
- **MCP → tool pool**: MCP tools attached via `assembleToolPool` as `mcp__server__tool` with preserved built-in-prefix ordering (prompt-cache-load-bearing).
- **State vs Config vs Settings three-layer split**: bootstrap-isolated process singleton (`STATE`) + `~/.claude.json` with write-through cache (`Config`) + 5-source Zod-validated read-only merge (`Settings`). Policy > flag > local > project > user.
- **Worktree chdir mid-setup** (`setup.ts:176-285`) mutates cwd/projectRoot, clears memory-file caches, re-captures hooks snapshot — everything downstream reads the new path.

**Independent within cluster?** No — each depends on ≥2 others. Document as a single "Bootstrap" section with a phased timeline diagram.

**Primary entry points**: `main.tsx:1903-1935` (`setup()` call site), `main.tsx:2573` (full env apply), `interactiveHelpers.tsx:100` (deferred prefetches).

---

## 4. Cluster B — Request Cycle (Reports 06–11)

**Members**: 06 REPL/input, 07 query orchestration, 08 prompt construction, 09 API interaction, 10 response parsing, 11 inner agent loop.

**Why they cluster**: they implement the tightest loop in the system — user prompt → query → API → stream → tool → observation → next turn. Each is directly called by its neighbor; data flow is linear with the inner agent loop as the decision hub.

**Shared data flow**:
```
REPL.tsx onQuery
   ↓
query() → queryLoop()  ← 10+ decision branches at iteration end
   ↓
  pre-API pipeline (5-way compaction stack)
   ↓
  getSystemPrompt + prependUserContext
   ↓
  deps.callModel → queryModel (raw stream, idle watchdog)
   ↓
  per-block streaming accumulator (one AssistantMessage per content_block_stop)
   ↓
  StreamingToolExecutor.addTool (mid-stream) + getCompletedResults
   ↓
  needsFollowUp? loop : handleStopHooks + return Terminal
```

**Critical interactions**:
- **QueryGuard single-serialization** (REPL) ↔ **queryLoop State machine** (query): one-query-at-a-time enforced in REPL, per-iteration State rebuilt at `continue` sites in query.
- **Content-presence over stop_reason**: the loop's sole exit signal is `needsFollowUp` — flipped when any tool_use block arrives. `stop_reason === 'tool_use'` documented-unreliable.
- **Streaming fallback dance**: non-streaming fallback in claude.ts ↔ tombstoning in query.ts ↔ executor `.discard()` + rebuild — invariants cross file boundaries.
- **Mid-stream tool dispatch**: `StreamingToolExecutor` starts tools at `content_block_stop` before message completes; concurrent-safe parallel; siblingAbortController cascades Bash errors only.
- **Prompt-cache disciplined across**: system-prompt boundary marker, tool prefix ordering, byte-exact text preservation, sticky beta-header latches (cross-cuts to Cluster A's State).
- **Withheld errors**: 413/media/max_output_tokens held during streaming, replayed only post-needsFollowUp check.
- **Thinking signature model-binding**: stripped before fallback on ant builds; tombstoned on streaming fallback.

**Internal sub-structure** (for the final report):
- **B1 Input & serialization**: REPL, PromptInput, QueryGuard, slash-command split (pre-loop vs in-loop).
- **B2 Context preparation**: 5-way compaction stack (snip → microcompact → collapse → autocompact → reactiveCompact), userContext prepend, system prompt boundary.
- **B3 Streaming pipeline**: raw stream open, idle watchdog, per-block accumulator, direct-mutation invariant, stop-reason recovery.
- **B4 Tool dispatch & observation**: executor concurrency model, per-tool state machine, permission + hook interleaving, tool-result merge.
- **B5 Decision hub**: 10+ decision branches, withheld error replay, stop-hook handling, token-budget continuation.

**Document as**: one "Request Cycle" section with B1–B5 sub-sections and a single choreography diagram spanning all six reports.

---

## 5. Cluster C — Lifecycle (Reports 13, 14, 15)

**Members**: 13 interruption handling, 14 session persistence, 15 cleanup/exit.

**Why they cluster**: all three participate in the session end-of-life. Signal → AbortController cascade (13) feeds graceful shutdown (15), which triggers final flush + metadata re-append (14). They share the `cleanupRegistry` dispatch surface and time-bounded async discipline.

**Shared choreography on exit**:
```
OS signal (or Ctrl+D, /exit)
   ↓
gracefulShutdown(code, reason)
   ↓  [sync-first phase]
cleanupTerminalModes + printResumeHint (writeSync, before await)
   ↓  [async phases, time-bounded]
runCleanupFunctions (2s cap) ─▶ Project.flush + reAppendSessionMetadata ← persistence
                                MCP stdio SIGINT→SIGTERM→SIGKILL (600ms)
                                Watchers · locks · telemetry flush
   ↓
executeSessionEndHooks (1.5s default)
   ↓
profileReport · tengu_cache_eviction_hint
   ↓
analytics flush (500ms cap)
   ↓
forceExit(code) — SIGKILL fallback on EIO
```

**Critical interactions**:
- **Signal → AbortController → withRetry**: `signal.aborted` checked in query.ts and withRetry.ts hot paths; abort reasons (`user-cancel`/`background`/`interrupt`) dispatched semantically.
- **Failsafe budget scales with hook budget**: `max(5000, sessionEndTimeoutMs + 3500)` — fix for gh-32712 silent-truncation.
- **Persistence invariants load-bearing for cleanup**: `reAppendSessionMetadata` must run *after* flush (64KB tail window); `ai-title` never re-appended; metadata pre-boundary recovery via `scanPreBoundaryMetadata`.
- **Conversation recovery ↔ resume**: `--continue`/`--resume`/`--fork-session` flows described in Persistence; `processSessionStartHooks('resume', …)` bridges to bootstrap (Cluster A) on restart.
- **Tombstoning double role**: partial-message tombstoning in streaming fallback (Request Cycle) + transcript tombstoning via `removeMessageByUuid` (Persistence) are distinct but share the "invalidate to avoid replay of stale state" pattern.

**Independent within cluster?** Partial — Persistence has standalone load/save pipelines, but its cleanup-hook integration is tight. Document as:
- **C1 Cancellation taxonomy**: three layers (signal/abort/retry), semantic reasons, abort-tree with WeakRef.
- **C2 Retry and fallback** (`withRetry.ts`): foreground/background 529 triage, persistent mode, fast-mode, Opus fallback, provider-specific auth recovery.
- **C3 Persistence write path**: JSONL append-only, write queue, materialization, 64KB tail invariant, re-append discipline, tombstone.
- **C4 Persistence load path**: 5MB threshold, walkChainBeforeParse, applyPreservedSegmentRelinks, applySnipRemovals, resume/continue/fork flows.
- **C5 Shutdown pipeline**: 13-step sync-then-async, time-bounded phases, forceExit SIGKILL fallback, ordering rationale.

---

## 6. Cluster D — Background & Alternative Modes (Reports 12, 16, 17)

**Members**: 12 concurrent processes, 16 daemon mode, 17 bridge/templates.

**Why they cluster**: all describe work happening *outside* the main REPL request cycle — parallel prefetches, long-lived peers, and alternative process topologies (daemon-supervised workers, bridge/remote-control entrypoints). They share the `cleanupRegistry`, `concurrentSessions` registry, and init-bypass philosophy.

**Shared patterns**:
- **Init-bypass for fast start**: bridge entrypoints + daemon workers skip `init.ts` and re-do only the narrow subset they need (`enableConfigs`, `initSinks`, CWD, trust check).
- **Three entrypoint shapes converge on one poll loop**: standalone (`claude remote-control`), REPL-hosted (`/remote-control`), daemon-worker (`runBridgeHeadless`) all end up in `runBridgeLoop` (v1) or v2 SSE+CCR transport.
- **Daemon ↔ bridge entanglement**: `runBridgeHeadless` is the only analyzable daemon worker body; daemon's `AuthManager` IPC shape is defined by bridge's `HeadlessBridgeOpts` contract.
- **SessionKind registry** (`concurrentSessions.ts`): `interactive | bg | daemon | daemon-worker`; PID files coordinate `claude ps`, supervision, resume.
- **`.unref()` discipline universal** across all background timers — none holds the process alive.

**Missing source** (flagged in both reports 16 and 17):
- `src/daemon/main.ts`, `src/daemon/workerRegistry.ts` — feature('DAEMON') gated
- `src/cli/handlers/templateJobs.js` — feature('TEMPLATES') gated
- Only the contract with these subsystems is analyzable.

**Document as**:
- **D1 Background ring around REPL**: six tiers of concurrent activity (pre-render, deferred, post-submit housekeeping, long-lived peers, per-turn transients, external peers), `.unref()` + housekeeping admission control.
- **D2 MCP/Bridge/Remote transports**: REPL-hosted alternative transports, v1 env-based vs v2 env-less split, auth dead-token cross-process backoff.
- **D3 Daemon architecture** (call-site level): supervisor + worker topology, error taxonomy (`BridgeHeadlessPermanentError` → `EXIT_CODE_PERMANENT` park vs transient respawn), shared `AuthManager` IPC, at-least-three worker kinds.
- **D4 Templates (contract-only)**: `cli.tsx:211-222` dispatch shape, `feature('TEMPLATES')` gate.

---

## 7. Independent Subsystems (no cluster membership)

After cluster assignment, every Wave 1 report is absorbed into A–D. **There are no true standalone reports**, but several subsystems *within* clusters can be documented independently as reference material without blocking cluster comprehension:

- **`withRetry.ts`** (822 LOC) — within Cluster C, but its state machine (4 auth refresh paths, fast-mode dual path, persistent mode heartbeat) is self-contained and worth a standalone reference table.
- **`query/transitions.ts` reasons enumeration** — within Cluster B, 7 Continue × 12 Terminal reasons form a pure state-machine reference.
- **Error taxonomy (`errors.ts`, ~30 codes)** — cross-cuts Clusters B+C but is flat reference material.
- **`EXIT_REASONS` enum + exit-code inventory** — within Cluster C, flat reference.
- **`SessionKind` enum + `concurrentSessions` PID file schema** — within Cluster D, flat reference.
- **Entry-type dispatch table in `appendEntry`** (18+ types) — within Cluster C4, flat reference.

These go into an **Appendix** of the final report rather than the narrative sections.

---

## 8. Cross-Cutting Concerns

Each of these threads through ≥3 clusters and should be lifted into dedicated cross-cutting sections rather than duplicated per cluster.

### CC-1: Trust Boundary
**Clusters touched**: A (establishes), B (prompt content + env vars), C (forceLogin invariants on resume), D (bridge policy check).

Single trust gate (`showSetupScreens`) unlocks: env-var application, OAuth helpers, telemetry, GitHub mapping, GrowthBook auth, managed-env apply. Home directory never persists trust; enterprise `forceLoginOrgUUID` uses uncached profile. Session flag + per-directory disk; parent-walk propagation; memoize-only-true.

### CC-2: Prompt Cache Discipline
**Clusters touched**: A (built-in tool prefix ordering, settings beta-header latches), B (system-prompt boundary marker, byte-exact text preservation), C (`/clear` and `/compact` clear latches).

One-boundary invariant, sticky beta-header latches, latch clears only via explicit user actions, text content byte-exact across normalization round-trip. Broken cache → every request uncached → cost + latency cliff.

### CC-3: Abort / Cancellation Tree
**Clusters touched**: A (teardown on setup failure), B (tool/API cancellation), C (shutdown entry), D (bridge/daemon worker abort).

Three layers (OS signals → AbortController tree with WeakRef → API retry loop). Semantic reasons (`user-cancel`/`background`/`interrupt`) dispatched via `signal.reason`. `isAbortError` triple-check (SDK class names minified).

### CC-4: Hooks Extension Surface
**Clusters touched**: A (PreToolUse, SessionStart at mount), B (PreToolUse/PostToolUse per tool, Stop hooks), C (SessionEnd hooks, `PermissionDenied`, `TaskCompleted`, `TeammateIdle`), D (file-watcher, settings/skill change).

15+ hook event types; hooks can rewrite inputs, override permissions, prevent continuation, retry on denial. MCP tool post-hook ordering differs (hooks rewrite output).

### CC-5: Subagent / Teammate Scoping
**Clusters touched**: B (`toolUseContext.agentId` gates filtering, async agent tool allowlist), C (per-agent persistence files, `TaskCompleted` hooks on subagent exit), D (teammate via InProcessTeammateTask, swarm backends).

Subagents force `shouldAvoidPermissionPrompts=true`; inherit parent mode + cwd; async agents exclude MCP tools; `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` allowlist.

### CC-6: Time-Bounded Async / `.unref()` Discipline
**Clusters touched**: B (idle watchdog 90s, stall 30s, stream timeout), C (2s registry, 1.5s SessionEnd, 500ms analytics, max(5s, hook+3.5s) failsafe), D (MCP SIGINT→SIGTERM→SIGKILL escalation 100→500→600ms, housekeeping admission control).

Every async phase has a cap; every timer `.unref()`'d. Nothing can hang the process. Sync-first for user-visible state before any `await`.

### CC-7: Post-Incident Fingerprints
Scattered across every cluster — production incidents whose invariants the current code is structured around:
- CC-34 (session ID + projectDir atomicity)
- gh-32712 (failsafe budget truncation)
- inc-3694, inc-4029, inc-4258 (streaming / parsing fallback)
- inc-3930 (tombstone OOM)
- #21056 (permission bubble-up for ExitPlanMode)
- #14373, #23537 (progress in chain)
- #22453 (resume truncation)
- ANT-344 (persistent-retry keep-alive TODO)
- adamr-20260320 (fork-session sessionId cross-contamination)
- gh-30217 (transcript_path cwd mismatch)

Worth a dedicated "Historical incidents shaping current invariants" appendix.

---

## 9. Final Report Section Plan

Proposed `codebase-report.md` structure:

```
1. Executive Summary
2. Runtime topology (one diagram: main process, daemon, bridge workers, MCP subprocesses, Ink tree)
3. Bootstrap (Cluster A)
    3.1 setup() phases
    3.2 Trust boundary and auth cascades
    3.3 Tool pool assembly
    3.4 MCP pool bootstrap
    3.5 State / Config / Settings three-layer model
4. Request Cycle (Cluster B)
    4.1 REPL input + QueryGuard
    4.2 Context preparation (compaction stack)
    4.3 Prompt construction + cache boundary
    4.4 API streaming + fallback
    4.5 Per-block parsing + tool dispatch
    4.6 Decision hub + stop-hook handling
5. Lifecycle (Cluster C)
    5.1 Cancellation taxonomy
    5.2 Retry / fallback (withRetry)
    5.3 Persistence write path
    5.4 Persistence load / resume / fork
    5.5 Shutdown pipeline
6. Background & Alternative Modes (Cluster D)
    6.1 Background ring (prefetches, housekeeping, watchers)
    6.2 MCP / Bridge / Remote transports
    6.3 Daemon architecture (contract-only)
    6.4 Templates (contract-only)
7. Cross-cutting concerns
    7.1 Trust boundary
    7.2 Prompt cache discipline
    7.3 Abort tree
    7.4 Hooks surface
    7.5 Subagent scoping
    7.6 Time-bounded async
    7.7 Post-incident invariants
8. Appendices (flat reference material)
    A. withRetry state machine
    B. Continue × Terminal reasons matrix
    C. Error taxonomy
    D. Exit codes + ExitReason enum
    E. SessionKind + PID registry schema
    F. JSONL Entry types
9. Recommendations for follow-up (feature-gated source, deep dives)
```

Every Wave 1 report maps into at least one numbered section. No orphaned reports. Cross-cutting concerns deliberately avoid duplication across sections 3–6.

---

## 10. Open Items for Compiler (task #23)

- Runtime topology diagram must span four process kinds (interactive, bg, daemon, daemon-worker) + external peers (MCP subprocesses, bridge/remote WS).
- Request Cycle choreography diagram should be a single figure spanning B1–B5.
- Include missing-source callouts for: `daemon/*`, `templateJobs.js`, `query/transitions.ts`, MCP connection-pool internals — all feature-gated or extraction artifacts (per `wave-1-review.md`).
- Cross-cutting concerns (§7) should use callouts back to cluster sections rather than inline duplication.
- Appendices (§8) should be tables-only, no prose.
- The "post-incident invariants" thread (§7.7 / CC-7) is the strongest evidence that this codebase is carefully engineered — worth foregrounding in the executive summary.
