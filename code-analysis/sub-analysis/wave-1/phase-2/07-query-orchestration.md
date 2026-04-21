# query.ts Orchestration & Async Patterns

Task #17 — analyze `src/query.ts` and the orchestration logic that drives the
agent loop. Focus here is the **async-generator pattern**, **yield discipline**,
**state-machine shape**, and **dependency-injection seams** that make this
1,729-line file tractable. The inside-a-turn think/act/observe cycle is covered
separately in `11-inner-agent-loop.md` — this report deliberately zooms out to
the orchestration shell so the two reports compose without overlap.

Files read (primary):

- `src/query.ts` (lines 1–1729, full)
- `src/query/deps.ts` (full, 40 lines)
- `src/query/config.ts` (full, 46 lines)
- `src/query/stopHooks.ts` (header, 80 lines)
- `src/query/tokenBudget.ts` (headers)

Adjacent signals grepped: `reason:` exit sites in `query.ts`, references to
`QueryDeps`, `./query/transitions.js` (imports `Terminal, Continue` — file not
present in extraction; types inferable from uses).

---

## 1. The generator surface: two generators stacked

Two async generators are exported/used together.

### 1.1 `query()` — the public thin wrapper

```ts
// src/query.ts:219–239
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // Only reached if queryLoop returned normally. Skipped on throw (error
  // propagates through yield*) and on .return() (Return completion closes
  // both generators). This gives the same asymmetric started-without-completed
  // signal as print.ts's drainCommandQueue when the turn fails.
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

The shape — `yield*` to the inner generator, then run cleanup only on
**normal return** — is the orchestration's one external lifecycle hook. It
deliberately asymmetrically fires `lifecycle: completed` only on normal
termination, mirroring `print.ts`'s `drainCommandQueue`. Callers that abort
the generator via `.return()` (REPL interrupt) or throw (model error) never
see `completed`, so consumers of the command-lifecycle stream can treat
"started without completed" as the failure signal.

### 1.2 `queryLoop()` — the state machine

```ts
// src/query.ts:241–251
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
> {
```

Return value is a `Terminal` discriminated union (not exported in the
extracted tree; `./query/transitions.js` is imported at `query.ts:104` but not
present — grep confirms no matching file). Its full shape can be reconstructed
from the nineteen `return { reason: ... }` sites (see §6).

---

## 2. The yield protocol: what flows out of the generator

Yield items union (lines 244–250):

| Kind | When yielded | Consumer behavior |
|---|---|---|
| `StreamEvent` | Incremental model output (deltas, tool_use blocks as they stream) | Rendered mid-turn in REPL; buffered by SDK consumers |
| `RequestStartEvent` | `{ type: 'stream_request_start' }` once per iteration (line 337) | Lets consumers reset per-request UI state |
| `Message` (assistant / user / attachment / system) | Completed API message, tool result, interruption, attachment batch | Persisted to log; becomes history input for next turn |
| `TombstoneMessage` | On streaming fallback — for every partial `assistantMessages` already yielded (line 716–718) | Consumers must drop tombstoned UUIDs from their UI and transcript. Specifically prevents "thinking blocks cannot be modified" API errors by purging invalid-signature partials |
| `ToolUseSummaryMessage` | From `pendingToolUseSummary` awaited at iteration top (line 1055–1060) — summary of *previous* turn's tool batch, surfaced here because the Haiku call overlaps with the current turn's ~30s model streaming |

The key **yield discipline** design: the generator is the single ordering
arbiter for the turn. Every side effect visible to the UI (system messages,
tombstones, attachments, tool results, recovery messages) goes through a
`yield` on this one generator. There is no parallel UI stream — consumers only
have to subscribe to one thing.

### 2.1 Withholding: intermediate errors that the generator *must not* yield

The streaming inner loop (lines 659–863) withholds four error classes instead
of yielding them:

```ts
// src/query.ts:799–822
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, isPromptTooLongMessage, querySource)) {
    withheld = true
  }
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) { withheld = true }
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) { withheld = true }
if (isWithheldMaxOutputTokens(message)) { withheld = true }
if (!withheld) { yield yieldMessage }
```

Rationale (lines 788–795, paraphrased from comment): withheld messages are
still pushed to `assistantMessages` so the post-stream recovery logic can find
them; only the **yield** is suppressed. Yielding a 413 or max-output-tokens
error early would leak "the model failed" to SDK consumers (cowork, desktop)
that terminate on any `error` field — but the recovery loop would keep running
and its successful retry would go unheard.

This withholding is the single hardest invariant in the file: the **withhold
predicate** (inside the stream loop) and the **recovery predicate** (after the
stream loop) must agree *byte-for-byte*. Line 622–627 explicitly hoists
`mediaRecoveryEnabled` once per turn to guarantee the same `CACHED_MAY_BE_STALE`
gate reading, because a flip mid-stream would cause the stream loop to withhold
while the recovery loop declines to act on the withheld error — eating it.
PTL doesn't hoist because its withholding is ungated and predates the experiment.

### 2.2 Tombstoning: unyielding already-yielded messages

Lines 712–723 (streaming fallback) and 899–903 (FallbackTriggeredError) both
yield `{ type: 'tombstone', message: msg }` for every assistant message
already yielded. This is a **reverse-yield**: the generator tells the consumer
"forget what I just told you." Used in two situations:

1. Streaming fallback mid-response (line 678–680, `streamingFallbackOccured`).
   Partial thinking blocks have signatures tied to the original model; replaying
   them against the fallback model produces `thinking blocks cannot be modified`
   API errors. Tombstoning forces both UI removal *and* transcript scrub.
2. `FallbackTriggeredError` caught at line 894. Here the stricter form
   `yieldMissingToolResultBlocks` fires instead (line 900), emitting synthetic
   `is_error: true` tool_results for any `tool_use` blocks in the abandoned
   assistant messages — since those tool_uses were already streamed to the
   consumer, they need matching tool_results or the transcript has orphan pairs.

---

## 3. The State struct pattern — why there are only 7 continue sites

```ts
// src/query.ts:204–217
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  // Why the previous iteration continued. Undefined on first iteration.
  // Lets tests assert recovery paths fired without inspecting message contents.
  transition: Continue | undefined
}
```

The `while (true)` loop body destructures `state` at the top of each
iteration (lines 308–321):

```ts
let { toolUseContext } = state            // reassigned within iteration
const {
  messages,
  autoCompactTracking,
  maxOutputTokensRecoveryCount,
  ...
} = state                                 // read-only between continue sites
```

Every `continue` site writes `state = { ...new values }` — nine related fields
in one assignment. Comment at line 265–267: *"Continue sites write
`state = { ... }` instead of 9 separate assignments."* This is a deliberate
refactor choice — the alternative (mutating 9 fields individually) is what the
comment at line 1292–1296 warns about:

```ts
// Preserve the reactive compact guard — if compact already ran and
// couldn't recover from prompt-too-long, retrying after a stop-hook
// blocking error will produce the same result. Resetting to false
// here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
hasAttemptedReactiveCompact,
```

A past bug where one iteration's continue forgot to propagate one field
caused production spirals. The State struct makes "forgot to set X" an
explicit choice — each continue is one literal object — rather than an implicit
omission.

### 3.1 The `transition` field

```ts
// Why the previous iteration continued. Undefined on first iteration.
// Lets tests assert recovery paths fired without inspecting message contents.
transition: Continue | undefined
```

The `Continue` tag set (grepped from `transition: { reason: ... }` sites):

- `collapse_drain_retry` (with `committed: number`)
- `reactive_compact_retry`
- `max_output_tokens_escalate`
- `max_output_tokens_recovery` (with `attempt: number`)
- `stop_hook_blocking`
- `token_budget_continuation`
- `next_turn`

These are the **seven continue reasons**. The `transition` field is only used
by tests (per the comment) to assert which recovery path fired without
inspecting yielded messages. It also serves as an **anti-loop guard** at line
1091–1092: the collapse-drain-retry path is skipped if the previous transition
was already `collapse_drain_retry` — one-shot per turn, then falls through to
reactive compact.

---

## 4. The QueryDeps injection seam

```ts
// src/query/deps.ts:21–31
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming       // the API call
  microcompact: typeof microcompactMessages       // cache-edit or full rewrite
  autocompact: typeof autoCompactIfNeeded          // 80% threshold summary
  uuid: () => string                               // randomUUID()
}
```

`queryLoop()` reads `const deps = params.deps ?? productionDeps()` at line 263.
Tests override these via `QueryParams.deps` instead of the
`import-module-and-spy` boilerplate that predates this pattern.

The deliberate narrowness is documented at `deps.ts:19–20`:
> "Scope is intentionally narrow (4 deps) to prove the pattern. Followup
>  PRs can add runTools, handleStopHooks, logEvent, queue ops, etc."

So `runTools`, `handleStopHooks`, queue operations remain module-scoped imports
— they're mockable via `vi.spyOn` but not yet first-class deps. This is an
**in-progress abstraction**: the seam exists but hasn't been extended.

### 4.1 `QueryConfig` — the immutable snapshot companion

```ts
// src/query/config.ts:15–27
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

Built once at query entry (line 295, `const config = buildQueryConfig()`) and
never reassigned. The comment at `config.ts:11–14` says the goal is
**step() extraction tractability**:

> "Separating these from the per-iteration State struct and the mutable
>  ToolUseContext makes future step() extraction tractable — a pure reducer
>  can take (state, event, config) where config is plain data."

So the file is inching toward a Redux-like shape: `(State, Event) → State`
(plus yielded Action). Three inputs are now formally separated:
`State` (mutable, per-iteration), `QueryConfig` (immutable, per-query),
`ToolUseContext` (mutable, shared across the agent tree).

**Feature gates are intentionally excluded** (`config.ts:14`): `feature()` is a
bun:bundle tree-shaking boundary and must stay inline so dead-code elimination
triggers. A `QueryConfig.features.DAEMON` field would defeat DCE because the
bundler can't prove all reads at build time.

---

## 5. The `using` disposable pattern

```ts
// src/query.ts:301–304
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages,
  state.toolUseContext,
)
```

TC39 explicit resource management. Disposes on **every** generator exit path
— `return`, `throw`, caller's `.return()`, caller's `.throw()`. The comment
(line 299–301):

> "Fired once per user turn — the prompt is invariant across loop iterations,
>  so per-iteration firing would ask sideQuery the same question N times.
>  Consume point polls settledAt (never blocks). `using` disposes on all
>  generator exit paths — see MemoryPrefetch for dispose/telemetry semantics."

Two design choices visible here:

1. **Positioning** — this is declared *before* the `while (true)` so the
   prefetch runs during the first model stream; per-iteration declaration
   would restart the side-query each turn.
2. **Non-blocking consume** (line 1599–1614) — polls `settledAt !== null` and
   only consumes if settled; if not, it skips and tries again next iteration.
   The comment at 1594–1598 notes `consumedOnIteration` is a cumulative filter:
   memories the model already Read/Wrote/Edited on earlier iterations are
   filtered via `readFileState`, which per-iteration `toolUseBlocks` alone
   couldn't see.

This is the only `using` declaration in the file and fits the narrow use case
— a single long-lived resource whose cleanup must survive abnormal generator
termination.

---

## 6. The Terminal taxonomy — 12 exit reasons

All `return { reason: ... }` sites in `queryLoop()`:

| Exit reason | Source line | Trigger |
|---|---|---|
| `blocking_limit` | 646 | Hard token-limit preempt when auto-compact off |
| `image_error` | 977, 1175 | ImageSizeError/ImageResizeError; withheld media with no recovery |
| `model_error` | 996 | Caught error from streaming (with `error` field) |
| `aborted_streaming` | 1051 | Abort during model stream |
| `prompt_too_long` | 1175, 1182 | 413 with no recovery path |
| `completed` | 1264, 1357 | Normal turn end (no follow-up) |
| `stop_hook_prevented` | 1279 | Hook set preventContinuation |
| `aborted_tools` | 1515 | Abort during tool execution |
| `hook_stopped` | 1520 | `hook_stopped_continuation` attachment |
| `max_turns` | 1711 | turnCount > maxTurns |

12 distinct terminal states × 7 continue reasons = 19 possible transitions out
of any iteration. The `switch (terminal.reason)` handling lives outside this
file — callers in REPL.tsx (interactive) and print.ts (non-interactive)
specialize on these tags.

### 6.1 Exit paths that bypass stop hooks

Three places (lines 1173–1175, 1180–1183, 1262–1265) explicitly early-return
without calling `handleStopHooks`. The repeated comment justifies it identically:

> "Do NOT fall through to stop hooks: the model never produced a valid response,
>  so hooks have nothing meaningful to evaluate. Running stop hooks on
>  prompt-too-long creates a death spiral: error → hook blocking → retry →
>  error → … (the hook injects more tokens each cycle)."

This is a **load-bearing comment** — a historical incident that shaped
control flow. `executeStopFailureHooks` is called instead (fire-and-forget) so
observability hooks still see the failure.

---

## 7. Streaming vs non-streaming tool execution

```ts
// src/query.ts:1380–1382
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

The fork point at the **post-stream dispatch**. `StreamingToolExecutor`
(instantiated at line 562–568, gated on `config.gates.streamingToolExecution`
from Statsig `tengu_streaming_tool_execution2`) receives tool_use blocks
*during* model streaming:

```ts
// src/query.ts:841–844
for (const toolBlock of msgToolUseBlocks) {
  streamingToolExecutor.addTool(toolBlock, message)
}
```

…which lets the executor start running tools *before* the stream completes.
Completed results (line 851) are yielded mid-stream. After the stream,
`getRemainingResults()` drains anything still in flight.

Non-streaming path (`runTools`) waits for the full response, then dispatches
sequentially/in-parallel based on block content. `runTools` lives at
`src/services/tools/toolOrchestration.js` and is the older path.

**Why the streaming path is gated**: the fallback logic (line 712–741)
and abort handling (line 1015–1029) both have to discard partial executor state:

```ts
// src/query.ts:733–740
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(
    toolUseContext.options.tools, canUseTool, toolUseContext,
  )
}
```

The streaming executor is a stateful workhorse with non-trivial lifecycle
(add during stream, complete during stream, drain after stream, discard on
fallback, generate-synthetic-results on abort). The Statsig gate lets the
team roll it back per-account if a new edge case surfaces.

---

## 8. The pre-API pipeline — five stages before `callModel`

Each iteration top (lines 365–648) runs five gated transforms, in order:

1. **`applyToolResultBudget`** (line 379–394) — per-message budget enforcement
   on aggregate tool result size. Runs *before* microcompact because cached MC
   matches by `tool_use_id` and ignores content, so the two compose cleanly.
   Persists `contentReplacementState` to sidechain (agent) or session
   (repl_main_thread) files, but *not* for ephemeral forked agents
   (agent_summary etc.) — line 376–378.

2. **`snipCompactIfNeeded`** (line 401–410, gated `HISTORY_SNIP`) — in-place
   history snip. `snipTokensFreed` is plumbed through so autocompact's
   threshold check reflects what snip removed; `tokenCountWithEstimation`
   alone can't see it because it reads usage from the preserved tail.

3. **`deps.microcompact`** (line 413–426) — cache-edit (CACHED_MICROCOMPACT) or
   full rewrite. Cache-edit returns `pendingCacheEdits` which the *post-stream*
   code (line 870–892) uses to emit a boundary message with the actual API
   `cache_deleted_input_tokens` rather than a client-side estimate.

4. **`contextCollapse.applyCollapsesIfNeeded`** (line 440–447, gated
   `CONTEXT_COLLAPSE`) — commits more collapses if needed. The comment at
   432–439 explains why this runs *before* autocompact:

   > "if collapse gets us under the autocompact threshold, autocompact is a
   >  no-op and we keep granular context instead of a single summary"

   Nothing is yielded — the collapsed view is a read-time projection over the
   REPL's full history. Summary messages live in the collapse store, not the
   REPL array. `projectView()` replays the commit log on every entry.

5. **`deps.autocompact`** (line 454–467) — 80%-threshold summary. Returns
   `compactionResult` on success or `consecutiveFailures` for the circuit breaker.

6. **Blocking-limit preempt** (line 628–648) — only when autocompact is OFF
   *and* neither reactive compact nor context collapse is active. Synthetic
   `PROMPT_TOO_LONG_ERROR_MESSAGE` is yielded, returning `blocking_limit`
   without ever calling the API. Multi-level skip logic documented at 604–620:
   reactive compact's preempt would starve if a synthetic error fired first;
   collapse's `recoverFromOverflow` drains staged collapses on a *real* 413,
   so a synthetic one here would skip both recovery paths.

This is the **context-management pipeline**: each stage is a bounded transform
producing `messagesForQuery`. By the time `callModel` is invoked (line 659), the
history has been through size-trimming, snip, microcompact, collapse, autocompact,
and blocking-limit guard.

---

## 9. Recursive continuation — the shape of "the next turn"

```ts
// src/query.ts:1714–1727
queryCheckpoint('query_recursive_call')
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,
  stopHookActive,
  transition: { reason: 'next_turn' },
}
state = next
```

This is **not recursion via `yield* query()`**. The old pattern (now replaced)
recursively awaited a fresh generator. Current design is flat: the `while`
loop continues with a new `state`, and the same generator frame yields for
every turn of the whole agent trajectory.

Benefits of this flattening:

- **Single `using` scope** — one memory prefetch lifetime covers the whole turn
  trajectory, not per-recursion.
- **No stack depth** — a 50-turn trajectory is still one generator frame.
- **Cleaner abort** — one `.return()` cancels everything; no "which frame?" question.
- **`consumedCommandUuids` is a single array** — appended across iterations,
  flushed once at the outer `query()` wrapper.

The `next_turn` transition carries forward minimal state: previous messages
plus new `assistantMessages` plus `toolResults`. `pendingToolUseSummary` is
passed through so the next iteration's top can await it (line 1055–1060).
`stopHookActive` is preserved so a blocking-error retry doesn't fall into
infinite stop-hook execution.

Fields **reset** on `next_turn` are explicit: `maxOutputTokensRecoveryCount: 0`,
`hasAttemptedReactiveCompact: false`, `maxOutputTokensOverride: undefined`.
Each recovery path is scoped to one turn — if the next turn hits the same
error, the recovery plays again from scratch (within its own exhaustion
counter).

---

## 10. Error taxonomy and interaction

Three error classes have first-class handling:

### 10.1 `FallbackTriggeredError` (line 894–951)

Thrown by `withRetry.ts` inside `deps.callModel`. Carries `originalModel` and
`fallbackModel`. Handler:

1. Sets `currentModel = fallbackModel`, `attemptWithFallback = true`
2. Yields synthetic tool_result blocks for each orphaned tool_use via
   `yieldMissingToolResultBlocks` (so transcript has matching pairs)
3. Clears `assistantMessages`, `toolResults`, `toolUseBlocks`
4. Discards+reconstructs `streamingToolExecutor` (line 912–919)
5. Sets `toolUseContext.options.mainLoopModel = fallbackModel` — observable
   to hooks and UI
6. Strips signature blocks via `stripSignatureBlocks(messagesForQuery)` for
   `USER_TYPE === 'ant'` only. Line 924–929:

   > "Thinking signatures are model-bound: replaying a protected-thinking
   >  block (e.g. capybara) to an unprotected fallback (e.g. opus) 400s.
   >  Strip before retry so the fallback model gets clean history."

7. Yields a `warning`-level system message so users see the switch
8. `continue` — retries the same iteration with new model, fresh state

### 10.2 `ImageSizeError` / `ImageResizeError` (line 970–978)

Both surface as `{ reason: 'image_error' }` with the error's message yielded
as an API-error assistant message. No retry — the next user turn must drop
or resize the offending image.

### 10.3 Generic stream errors (line 955–997)

Catch-all. Emits missing tool_results via `yieldMissingToolResultBlocks`,
yields an API-error assistant message with the real error text (line 986–989
explicitly rejects the old "[Request interrupted by user]" placeholder because
Node 18's missing `Array.prototype.with` was producing phantom interrupts that
masked real causes). Returns `{ reason: 'model_error', error }`.

### 10.4 `BridgeHeadlessPermanentError` (daemon worker only)

Not thrown from `query.ts` directly — comes from `bridgeMain.ts`. Relevant
here because `runBridgeHeadless` (the daemon worker) wraps a call path that
eventually invokes `query()`, and **only** `BridgeHeadlessPermanentError`
causes the supervisor to park instead of respawn the worker. Normal errors
surfaced from `query.ts` (model_error, prompt_too_long, etc.) become regular
response payloads in the bridge protocol — they don't cross the worker/
supervisor boundary as thrown errors.

---

## 11. Checkpoints: the orchestration's observability plane

Every pipeline stage emits a `queryCheckpoint(name)` call:

| Checkpoint | Line | Span |
|---|---|---|
| `query_fn_entry` | 339 | Iteration entry |
| `query_snip_start/end` | 402/409 | Snip stage |
| `query_microcompact_start/end` | 413/426 | Microcompact |
| `query_autocompact_start/end` | 453/468 | Autocompact |
| `query_setup_start/end` | 560/580 | Tool setup |
| `query_api_loop_start` | 652 | Pre-API |
| `query_api_streaming_start/end` | 658/864 | Model stream span |
| `query_tool_execution_start/end` | 1363/1409 | Tools |
| `query_recursive_call` | 1714 | Next iteration setup |

`queryCheckpoint` (from `src/utils/queryProfiler.ts`) lets the profiler
decompose a turn's latency into named segments without instrumenting each
call site with timing. This is the hook that produces the per-turn
latency dashboards in Grafana — changing stage names breaks dashboard queries.

Also fires `logEvent('tengu_*')` analytics at every branch point (fallback
triggered, streaming executor used/not used, query error, orphaned messages
tombstoned, budget continuation, etc.) — ~15 distinct event names in this file.

---

## 12. Integration points — what calls `query()`?

External callers of the public `query()` generator (grep `query(`):

- `src/screens/REPL.tsx` — interactive path, consumes yields for UI render
- `src/print.ts` — non-interactive path, buffers yields for JSON/text output
- `src/services/forkedAgent/*` — sub-agent forks (Task tool, session_memory compact)
- `src/services/compact/*` — the compact agent itself uses `query()` to run
  the summarizer prompt

All four callers treat the generator identically: iterate, accumulate messages,
react to specific `Terminal.reason` values at end.

The agent-hierarchy relationship is encoded in `toolUseContext.agentId`:

- `agentId === undefined` ⇒ main thread
- `agentId === '<uuid>'` ⇒ sub-agent (Task tool spawn)

`query.ts:1419` gates tool-use summary generation behind `!toolUseContext.agentId`
— subagents don't surface in the mobile UI, so the Haiku summary call is wasted.
`query.ts:1687–1701` similarly gates periodic task summary generation.

The `querySource: QuerySource` param (lines from `src/constants/querySource.ts`)
is the other identity marker — values like `repl_main_thread`, `sdk`,
`agent:<name>`, `compact`, `session_memory`, `remote_control` —
used for eight different branch decisions: persist-content-replacement,
commands-queue-draining, blocking-limit-skip (compact/session_memory bypass),
isWithheldPromptTooLong filter, etc.

---

## 13. Why this file is orchestration, not implementation

`query.ts` is ~1,729 lines but contains almost no business logic of its own.
Every real behavior lives in a delegated module:

| Behavior | Delegate |
|---|---|
| Model API call | `deps.callModel` = `services/api/claude.ts` |
| Prompt construction | `prependUserContext`, `appendSystemContext` in `utils/api.ts` |
| Microcompact | `deps.microcompact` = `services/compact/microCompact.ts` |
| Autocompact | `deps.autocompact` = `services/compact/autoCompact.ts` |
| Reactive compact | `services/compact/reactiveCompact.ts` (lazy-required) |
| Context collapse | `services/contextCollapse/index.ts` (lazy-required) |
| History snip | `services/compact/snipCompact.ts` (lazy-required) |
| Tool execution | `services/tools/StreamingToolExecutor.ts` / `toolOrchestration.ts` |
| Stop hooks | `query/stopHooks.ts` → `utils/hooks.ts` |
| Post-sampling hooks | `utils/hooks/postSamplingHooks.ts` |
| Tool-use summary | `services/toolUseSummary/toolUseSummaryGenerator.ts` |
| Attachments | `utils/attachments.ts` |
| Memory prefetch | `utils/attachments.ts#startRelevantMemoryPrefetch` |
| Skill prefetch | `services/skillSearch/prefetch.ts` (lazy-required) |
| Task summary | `utils/taskSummary.ts` (lazy-required) |
| Token budget | `query/tokenBudget.ts` |
| Content replacement persistence | `utils/sessionStorage.ts` |

What `query.ts` itself does:

1. **Sequence** these delegates in the right order per iteration
2. **Thread state** between them (compactionResult feeds blocking-limit skip,
   snipTokensFreed feeds autocompact, mediaRecoveryEnabled feeds withholding)
3. **Yield discipline** — only one thing can own a given yield slot, and
   `query.ts` is that thing
4. **Error funneling** — many error types come in; six terminal shapes go out
5. **Transition bookkeeping** — 7 continue reasons, each with its own reset set
6. **Recovery orchestration** — withhold/retry/surface for PTL, MOT, media

This separation is what makes the file tractable: adding a new recovery path
means (a) add a `Continue` tag, (b) add a continue site, (c) add a withhold
predicate if applicable. Adding a new pipeline stage means inserting one more
transform between lines 365 and 648 with its own checkpoint pair.

---

## 14. Complex control flow that needs its own analysis (flags)

These sub-behaviors invoked from `query.ts` each merit a dedicated report and
should not be expanded inline here:

1. **`StreamingToolExecutor`** (`src/services/tools/StreamingToolExecutor.ts`)
   — the lifecycle contract (addTool / getCompletedResults / getRemainingResults /
   discard), the synthetic tool_result generation on abort, and the interaction
   with `canUseTool` permissions are non-trivial. Referenced across 7 call
   sites in `query.ts`.

2. **`reactiveCompact.tryReactiveCompact`** (`src/services/compact/reactiveCompact.ts`)
   — the strip-retry for media errors, the `hasAttempted` spiral guard, the
   `cacheSafeParams` forked-agent invocation contract. Behavior at lines
   1119–1166 merges compact output into `State` in ways that bypass the normal
   pipeline.

3. **`contextCollapse.recoverFromOverflow`** (`src/services/contextCollapse/index.ts`)
   — staged-collapses drain semantics, the `committed` count that feeds the
   `collapse_drain_retry` transition, `projectView()` replay on entry.

4. **`handleStopHooks`** (`src/query/stopHooks.ts`) — 17KB file; nested
   generator yielding back through `yield*` at line 1267; the blocking-errors
   vs `preventContinuation` distinction.

5. **Queue drain + attachment injection** (lines 1546–1643) — agent-scoping
   rules (main vs subagent vs slash-commands vs prompt-mode-vs-task-notification
   mode), `getAttachmentMessages` side effects on readFileState, consumed-command
   filtering vs priority levels.

6. **Auto-compact threshold math** (`services/compact/autoCompact.ts`) —
   `calculateTokenWarningState`, blocking-limit math, 80% threshold, snip
   adjustment. Three different token-count reads (`tokenCountWithEstimation`,
   `finalContextTokensFromLastResponse`, `doesMostRecentAssistantMessageExceed200k`)
   each serve a distinct purpose.

7. **Task-budget (API-level)** (lines 197, 508–515, 1138–1146, 699–706) — distinct
   from the internal `tokenBudget` gate. Tracks cumulative pre-compact context
   windows and passes `remaining` to the server as a post-compact countdown.

---

## 15. Follow-ups / questions for the owning team

1. **The missing `./query/transitions.js` file** — `query.ts:104` imports
   `{ Terminal, Continue }` from a path that doesn't exist in the extracted
   tree. Either (a) `transitions.ts` is present but excluded from the extraction
   manifest, or (b) it was planned but not merged. Requires source re-extraction
   confirmation. (Same concern raised in `16-daemon-mode.md` §3 for `daemon/*`.)

2. **`QueryDeps` scope is advertised as "intentionally narrow (4 deps) to prove
   the pattern"** (deps.ts:19). How long has this pattern been in place, and
   has the follow-on expansion stalled? A larger `QueryDeps` would reduce the
   nine-ish `import-and-spyOn` boilerplates across `query.test.ts` that the
   comment describes.

3. **`buildQueryConfig()` doesn't include feature() gates** by design — DCE
   constraint. But the `step() extraction` goal requires pure-reducer
   testability, and feature gates branch heavily in `query.ts`. How does the
   team plan to reconcile these — a `config.features.*` that the reducer sees
   but the tree-shaker doesn't? A dual-build strategy?

4. **The 413 / media / MOT withholding invariant is stated in prose** (lines
   788–795, 622–627) but not enforced by type. A future `Withheld<E>` wrapper
   type that makes the withhold predicate and recovery predicate share a
   discriminant would move this from comment to compile-time.

5. **`FallbackTriggeredError`'s `stripSignatureBlocks` is gated on
   `USER_TYPE === 'ant'`** (line 927). External users running a capybara →
   opus fallback would still 400 on protected thinking blocks. Is this
   intentional (ants are the only ones with protected thinking access) or a
   missed case?

6. **The `using pendingMemoryPrefetch` is one of very few `using` declarations
   in the codebase.** The pattern is powerful — any I/O side-effect with
   async cleanup could use it — but adoption is limited. Worth a cross-codebase
   inventory to see where other long-lived resources could benefit (e.g. the
   `StreamingToolExecutor` lifecycle, which currently has explicit `discard()`
   calls at 3 sites that a `using` could unify).

7. **The `transition` field is "only for tests"** per the comment. A production
   path (e.g. logging the last-N transitions to a crash report on unexpected
   terminal) would justify keeping it; otherwise it's dead weight in the
   State struct. Worth checking telemetry coverage.

8. **The `TODO` at line 545** (`no need to set toolUseContext.messages during
   set-up since it is updated here`) points at an obsolete initialization
   path. Small cleanup but indicative — `toolUseContext.messages` vs
   `messagesForQuery` is a dual-source-of-truth hazard; the TODO implies the
   initial-set is redundant.

---

## Appendix A — The 14 `continue` sites → 7 transitions

| Site | Line | Transition reason | Resets |
|---|---|---|---|
| Post-compact continue | 535 (compact happens here, loop proceeds without continue — wraps over) | n/a (fallthrough) | tracking → new compacted state |
| Collapse drain retry | 1115 | `collapse_drain_retry` | transition only; preserves attempts |
| Reactive compact retry | 1165 | `reactive_compact_retry` | autoCompactTracking to undefined; hasAttemptedReactiveCompact → true |
| MOT escalate | 1220 | `max_output_tokens_escalate` | maxOutputTokensOverride=ESCALATED_MAX_TOKENS |
| MOT recovery | 1251 | `max_output_tokens_recovery` | appends recovery message; increments count |
| Stop-hook blocking | 1305 | `stop_hook_blocking` | MOTRecoveryCount=0; stopHookActive=true |
| Token budget continuation | 1340 | `token_budget_continuation` | appends nudge message; MOTRecoveryCount=0, hasAttemptedReactiveCompact=false |
| Next turn | 1727 | `next_turn` | MOTRecoveryCount=0, hasAttemptedReactiveCompact=false |

Terminal reasons (no continue): `blocking_limit`, `image_error` (2 sites),
`model_error`, `aborted_streaming`, `prompt_too_long` (2 sites), `completed`
(2 sites), `stop_hook_prevented`, `aborted_tools`, `hook_stopped`,
`max_turns`.

---

## Appendix B — Generator I/O surface summary

Input (`QueryParams`):

```
{
  messages, systemPrompt, userContext, systemContext,
  canUseTool, toolUseContext, fallbackModel?, querySource,
  maxOutputTokensOverride?, maxTurns?, skipCacheWrite?,
  taskBudget?: { total }, deps?
}
```

Output (per yield):

- `StreamEvent` — deltas during stream
- `RequestStartEvent` — once per API attempt
- `AssistantMessage` — completed API response (cloned with backfilled tool_use inputs for UI)
- `UserMessage` — tool results, recovery messages, interruptions
- `AttachmentMessage` — queued commands, memory, skill discovery, file changes
- `SystemMessage` — model-fallback warnings
- `TombstoneMessage` — unyield for orphaned partial messages
- `ToolUseSummaryMessage` — Haiku summary of previous turn's tool batch

Return (Terminal):

```
{ reason: 'completed' | 'blocking_limit' | 'prompt_too_long' | 'image_error'
        | 'model_error' | 'aborted_streaming' | 'aborted_tools' | 'max_turns'
        | 'stop_hook_prevented' | 'hook_stopped'
  // with variant-specific fields: { error } for model_error, { turnCount } for max_turns
}
```

The generator never `throw`s under normal operation — all errors funnel to a
yielded API-error message plus a terminal return, or they propagate to the
outer `try` at line 955 which converts them to `model_error`.
