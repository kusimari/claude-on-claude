# Inner Agent Loop: Think-Act-Observe Cycle

**Subsystem:** `inner-agent-loop` (task #3)
**Scope:** How the agent decides what to do next each turn — tool execution, result integration, continuation logic, stop conditions.
**Primary files:**
- `src/query.ts` (1729 lines) — the outer orchestrator / `queryLoop()` reducer
- `src/services/tools/toolOrchestration.ts` (189 lines) — batch partitioning between concurrent-safe and serial
- `src/services/tools/toolExecution.ts` (1745 lines) — per-tool validation → permission → execution → hooks
- `src/services/tools/StreamingToolExecutor.ts` (530 lines) — streaming-mode executor that starts tools during model streaming
- `src/query/stopHooks.ts` (473 lines) — Stop/SubagentStop/TaskCompleted/TeammateIdle hook execution and blocking-continuation logic
- `src/query/tokenBudget.ts` — token-budget continuation decision
- `src/query/deps.ts`, `src/query/config.ts` — I/O injection and immutable config snapshot

---

## 1. High-level shape of the loop

The think-act-observe cycle is implemented by `queryLoop()` in `src/query.ts:241-1728`, an `async function*` generator that yields stream/message events and returns a `Terminal` reason. The public entry point is `query()` at `src/query.ts:219-239`; it just delegates to `queryLoop()` and fires `notifyCommandLifecycle(..., 'completed')` for consumed queued commands when the loop returns normally (see comment at `src/query.ts:230-237` — skipped on throw and `.return()`).

At the top of each iteration the loop destructures an explicit mutable `State` struct (`src/query.ts:204-217`):

```ts
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
  transition: Continue | undefined   // why the previous iteration continued
}
```

The comment at `src/query.ts:265-279` explains the rationale: `toolUseContext` alone mutates mid-iteration; the rest is rebuilt at `state = { ... }` at each `continue` site so the 9+ recovery paths stay readable. A separate, immutable `QueryConfig` (`src/query/config.ts`) is snapshotted once per `query()` call for session ID + statsig/env gates — distinct from the per-iteration mutable `State`.

**One cycle per `while (true)` iteration.** The body follows the classic agent schema:

1. **Prepare context** — snapshot + microcompact + snip + autocompact + context-collapse apply before the API call (`src/query.ts:361-548`).
2. **Think** — stream from the model (`deps.callModel` at `src/query.ts:659-708`). Tool-use blocks are collected into `toolUseBlocks[]`; `needsFollowUp` flips true the moment any appears (`src/query.ts:832-835`).
3. **Act** — dispatch tools, either via the new `StreamingToolExecutor` (tools start executing while the model is still streaming) or via the serial/concurrent partitioned `runTools()` (`src/query.ts:1380-1408`).
4. **Observe** — tool results are appended to `toolResults[]`, merged with any queued-command attachments and memory/skill prefetches (`src/query.ts:1580-1628`).
5. **Decide** — stop (no tool_use), recover (withheld errors, max-output-tokens, stop-hook blocking), or continue (`src/query.ts:1062-1726`).

The loop exit is a `return { reason: ... }` returning a `Terminal` record. Observed terminal reasons (grep across `query.ts`): `completed`, `aborted_streaming`, `aborted_tools`, `blocking_limit`, `image_error`, `model_error`, `prompt_too_long`, `stop_hook_prevented`, `hook_stopped`, `max_turns`.

---

## 2. The single loop-exit signal: `needsFollowUp`

The sole signal the loop uses to decide "we are done thinking" is `needsFollowUp`, flipped true during streaming at `src/query.ts:829-835`:

```ts
const msgToolUseBlocks = message.message.content.filter(
  content => content.type === 'tool_use',
) as ToolUseBlock[]
if (msgToolUseBlocks.length > 0) {
  toolUseBlocks.push(...msgToolUseBlocks)
  needsFollowUp = true
}
```

The comment at `src/query.ts:552-557` is explicit about why the loop **does not** use the API's `stop_reason`:

> `stop_reason === 'tool_use' is unreliable -- it's not always set correctly. Set during streaming whenever a tool_use block arrives — the sole loop-exit signal. If false after streaming, we're done (modulo stop-hook retry).`

So the think→act transition is a pure *content inspection*: if the assistant message contains **any** `tool_use` block, act; otherwise enter stop-hooks + stop. Tool results (observations) are forwarded back into the next iteration's `messages` at `src/query.ts:1716`:

```ts
messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
```

This is the "observe-as-input" half of the cycle: results are *just more user messages* (as tool_result content blocks). There is no separate "observation" object in the state machine.

---

## 3. Pre-API context management (before "think")

Before the API call, the loop transforms `messagesForQuery` through a tightly ordered pipeline. Order matters — each stage's output feeds the next (`src/query.ts:365-548`):

| Order | Stage                       | Purpose                                                                                              | File                                          |
|------:|------------------------------|-------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| 1     | `getMessagesAfterCompactBoundary` | Drop anything before the last compact summary                                                         | `utils/messages.js`                           |
| 2     | `applyToolResultBudget`     | Aggregate-size cap on tool results; optionally persists replacement records                          | `utils/toolResultStorage.js`                  |
| 3     | `snipCompactIfNeeded`       | HISTORY_SNIP: remove individual oldest tool outputs when over a threshold                            | `services/compact/snipCompact.js`             |
| 4     | `deps.microcompact`         | Remove cached tool output fragments                                                                   | `services/compact/microCompact.js`            |
| 5     | `contextCollapse.applyCollapsesIfNeeded` | Projected view over collapse commit log (CONTEXT_COLLAPSE feature gate)                             | `services/contextCollapse/index.js`           |
| 6     | `deps.autocompact`          | Full-summary compaction if the post-snip/post-collapse count still exceeds threshold                 | `services/compact/autoCompact.js`             |
| 7     | Hard blocking-limit check    | Returns `{ reason: 'blocking_limit' }` if still over limit (skipped when compact/collapse is active) | `src/query.ts:615-648`                        |

The author-visible doctrine (`src/query.ts:396-399`): *snip before microcompact* (not mutually exclusive), *collapse before autocompact* (so collapse alone can avoid a full summary), and *reactive compact owns 413 recovery* (so proactive compact should defer when reactive is enabled — see the `collapseOwnsIt` walrus dance at `src/query.ts:615-620`).

Two callable seams are broken out via `QueryDeps` (`src/query/deps.ts`) — `callModel`, `microcompact`, `autocompact`, `uuid` — so tests can inject fakes without `spyOn`-per-module. The production factory `productionDeps()` just wires the real implementations. The comment calls this "Scope is intentionally narrow (4 deps) to prove the pattern" — look for follow-up PRs adding `runTools`, `handleStopHooks`, etc.

---

## 4. Streaming and tool dispatch ("act")

Two execution backends exist and the choice is made per-iteration via a statsig gate (`src/query.ts:561-568`):

```ts
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(toolUseContext.options.tools, canUseTool, toolUseContext)
  : null
```

### 4a. Streaming mode (`StreamingToolExecutor`)

Tools start running *while the model is still producing the rest of the message*. Inside the stream loop (`src/query.ts:847-862`), each tool_use block is handed to `streamingToolExecutor.addTool(toolBlock, message)` as it arrives, and already-completed results are drained into `toolResults[]` every iteration of the stream loop.

The class itself (`src/services/tools/StreamingToolExecutor.ts:40-`):
- Tracks each tool in a `TrackedTool` record with status `queued | executing | completed | yielded`.
- Creates a `siblingAbortController` that is a **child of** the parent abort (`createChildAbortController`, `StreamingToolExecutor.ts:59`). Firing this kills sibling subprocesses on Bash errors without ending the turn.
- Has a `discard()` method called on streaming fallback (`src/query.ts:733-740`) to abandon in-flight tool results before the fallback-model attempt starts.
- `getRemainingResults()` is consumed after streaming ends (`src/query.ts:1381`) to collect everything still in-flight or queued.

The fallback handling at `src/query.ts:712-741` (streaming fallback) and `src/query.ts:893-953` (`FallbackTriggeredError`) shows the careful choreography: on fallback, assistant messages are tombstoned (their thinking-block signatures are bound to the original model and would fail validation on retry — see the rule at `src/query.ts:151-162`, "the rules of thinking"), the executor is discarded+rebuilt, tool-result clearing happens, and the loop retries the same messages with the fallback model. Thinking-block signature stripping is Ant-only (`src/query.ts:927-929`).

### 4b. Non-streaming mode (`runTools`)

After streaming ends, if the streaming executor wasn't used, dispatch goes through `runTools()` (`src/services/tools/toolOrchestration.ts:19-82`). Its key contribution is **partitioning**: consecutive `isConcurrencySafe: true` tools are grouped into one batch and run in parallel; any non-safe tool becomes its own singleton batch and runs serially (`partitionToolCalls` at `toolOrchestration.ts:91-116`).

Concurrent batches go through `runToolsConcurrently` (`toolOrchestration.ts:152-177`), which uses `all()` (`utils/generators.js`) with a concurrency cap from `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10, `toolOrchestration.ts:8-12`).

**Context modifiers under concurrency.** Because tool calls may want to modify the shared `ToolUseContext` (e.g. expanding the additionalWorkingDirectories set), concurrent-safe tools queue their `modifyContext` callbacks per tool-use-ID and apply them *after* the whole concurrent batch completes (`toolOrchestration.ts:42-63`). Serial tools apply modifiers immediately. This is how the system avoids interleaved mutations during parallel execution.

### 4c. `runToolUse` — the per-tool state machine

The per-tool flow (`src/services/tools/toolExecution.ts:337-490`) is:

1. **Tool lookup** — first `findToolByName(options.tools, ...)`, then a fallback to `getAllBaseTools()` but only if the matched tool has the name in its `aliases` array (`toolExecution.ts:350-356`). This is how deprecated names like `KillShell → TaskStop` still work on old transcripts without leaving them in the active tool list.
2. **Abort check** — if the parent abort has already fired, yield a `CANCEL_MESSAGE` tool_result with `withMemoryCorrectionHint` and bail (`toolExecution.ts:415-453`).
3. **`streamedCheckPermissionsAndCallTool`** — wraps progress + final result into one async iterable via a `Stream<MessageUpdateLazy>` helper (`toolExecution.ts:492-570`).
4. **Input validation** — zod `safeParse`, then `tool.validateInput?.()`. Zod failures emit a `formatZodValidationError` tool_result; if the tool is a deferred tool that wasn't in the discovered set, `buildSchemaNotSentHint` appends guidance telling the model to call `TOOL_SEARCH_TOOL_NAME` first (`toolExecution.ts:614-733`).
5. **Speculative bash classifier** — for `BASH_TOOL_NAME`, `startSpeculativeClassifierCheck` is kicked off in parallel with the hooks and permission dialog setup (`toolExecution.ts:740-752`).
6. **PreToolUse hooks** — `runPreToolUseHooks` may emit messages, update input (`hookUpdatedInput`), set a hook permission decision, or `stop` the tool outright (`toolExecution.ts:800-862`).
7. **Permission resolution** — `resolveHookPermissionDecision` merges the hook's decision (if any) with `canUseTool` (`toolExecution.ts:921-931`). If not `allow`, emit a rejection tool_result with `ErrMsg`; if the denial came from the transcript auto-mode classifier, run `PermissionDenied` hooks and if any returns `retry: true` append a meta-message telling the model it may retry (`toolExecution.ts:995-1103`).
8. **Tool call** — `tool.call(callInput, {...ctx, toolUseId}, canUseTool, assistantMessage, progress => ...)` (`toolExecution.ts:1207-1222`). Note the `callInput` vs `processedInput` split at `1181-1205` — backfill-expanded inputs must not reach `tool.call()` if no hook replaced them, because tool results embed paths verbatim and changing them would break VCR transcript hashes.
9. **PostToolUse hooks** — ordering diverges for MCP tools (they read hook-rewritten output, so `addToolResult(toolOutput)` happens *after* the hook loop at `toolExecution.ts:1540-1542`); non-MCP tools get the pre-mapped result block immediately then run hooks afterward.
10. **Error path** — `McpAuthError` flips the client state to `needs-auth` via `setAppState` (`toolExecution.ts:1601-1629`); `PostToolUseFailure` hooks run; the final `tool_result` carries `is_error: true` and `formatError(error)` as content.

---

## 5. Observe and merge ("observe")

After the executor drains its `MessageUpdate` stream (`src/query.ts:1384-1408`), each `update.message` is yielded and any user-type normalized form is appended to `toolResults[]`. `update.newContext` updates `updatedToolUseContext`, so serial tools see each other's context edits.

Immediately after tool execution (before the next think), these are merged in order (`src/query.ts:1580-1628`):

1. **Attachments from queued commands** — `getCommandsByMaxPriority('later')` or `'next'` if `SleepTool` ran (`src/query.ts:1566-1578`). Slash commands are excluded (they must go through `processSlashCommand` after turn end); main-thread vs. subagent routing is by `agentId` — subagents only drain `task-notification` messages addressed to them.
2. **File-change attachments from `getAttachmentMessages`**.
3. **Memory prefetch results** — `startRelevantMemoryPrefetch` was fired once per user turn (`src/query.ts:301-304`); its result is consumed non-blockingly here if settled. `readFileState` is tracked across iterations to filter out memories the model already Read/Wrote/Edited.
4. **Skill discovery prefetch** — `collectSkillDiscoveryPrefetch` (gated on `EXPERIMENTAL_SKILL_SEARCH`). Ran in parallel with streaming and tool execution; the ~98% "hidden by main turn" stat in the comment at `src/query.ts:1620-1622` is the win.

After merging, consumed commands are removed from the queue and `notifyCommandLifecycle(uuid, 'started')` is fired (`src/query.ts:1630-1643`).

The merged `toolResults` plus `assistantMessages` are appended to `messages` at the bottom of the iteration and the loop continues.

---

## 6. Stop conditions (`needsFollowUp === false` branch)

When the assistant produced no tool_use blocks, the loop runs a multi-stage stop analysis (`src/query.ts:1062-1358`). The stages fire in order and each can cause a `continue` (recovery) or `return` (terminal):

### 6.1. Withheld prompt-too-long (413) recovery

During streaming, prompt-too-long errors are **withheld** (`src/query.ts:799-825`) so SDK callers don't terminate on a recoverable intermediate. The reactive-compact module (`reactiveCompact.isWithheldPromptTooLong`) and context-collapse module (`contextCollapse.isWithheldPromptTooLong`) both contribute to the `withheld` boolean. On `needsFollowUp === false`:

1. **Collapse-drain retry first** — if not already `collapse_drain_retry` transition (`src/query.ts:1089-1117`), call `contextCollapse.recoverFromOverflow` and if it committed anything, `continue` with `transition: { reason: 'collapse_drain_retry' }`.
2. **Reactive-compact retry** — on 413 or media-size error, call `reactiveCompact.tryReactiveCompact`. If it returns a compaction, replace `messages` with `postCompactMessages`, set `hasAttemptedReactiveCompact: true`, and `continue` (`src/query.ts:1119-1166`).
3. **Exhausted** — surface the withheld error, run `executeStopFailureHooks`, return. Explicit non-falltrough to stop hooks (`src/query.ts:1169-1183`): running stop hooks on a prompt-too-long creates a death spiral because the hook injects more tokens.

### 6.2. Withheld max-output-tokens recovery

(`src/query.ts:1188-1256`) Two-stage retry:

1. **Escalate** — if statsig gate `tengu_otk_slot_v1` is on, `maxOutputTokensOverride` is undefined, and no env override, retry the same request at `ESCALATED_MAX_TOKENS` (64k) without any meta message (`src/query.ts:1199-1221`).
2. **Resume** — up to `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` (`src/query.ts:164`), inject a meta user-message telling the model to "Resume directly — no apology, no recap of what you were doing" and continue.
3. **Exhausted** — yield the withheld error as-is.

### 6.3. Stop hooks and their recursion trap

`handleStopHooks` (`src/query/stopHooks.ts:65-472`) is the main post-turn gate. It:

- Saves `cacheSafeParams` via `saveCacheSafeParams` (main-session only) for `/btw` and side_question SDK control_request consumers (`stopHooks.ts:92-98`).
- Fires template-job classification with a 60-s timeout if `CLAUDE_JOB_DIR` is set (`stopHooks.ts:108-132`).
- Fires unrelated background bookkeeping unless `--bare`/SIMPLE: `executePromptSuggestion`, `executeExtractMemories`, `executeAutoDream` (`stopHooks.ts:136-157`).
- Chicago-MCP cleanup on main thread (`stopHooks.ts:164-173`).
- Runs `executeStopHooks` — can yield progress, block with errors, or set `preventContinuation` (`stopHooks.ts:180-295`).
- For teammates (subagents run via the teammate system, `isTeammate()` at `stopHooks.ts:335`), additionally runs `TaskCompleted` hooks for in-progress tasks owned by this teammate and `TeammateIdle` hooks (`stopHooks.ts:335-452`).

Stop-hook feedback in the outer loop (`src/query.ts:1267-1306`):

- `preventContinuation: true` → terminal `{ reason: 'stop_hook_prevented' }`.
- `blockingErrors.length > 0` → continue with blocking errors appended to `messages`, **`stopHookActive: true`**, and crucially `hasAttemptedReactiveCompact` preserved (not reset). The comment at `src/query.ts:1292-1296` documents an infinite-loop incident: resetting the guard caused `compact → still too long → error → stop hook blocking → compact → …` burning thousands of API calls.

### 6.4. Token-budget continuation

(`src/query.ts:1308-1355`, `src/query/tokenBudget.ts`) Gated on `feature('TOKEN_BUDGET')`. Only runs on main-thread (subagents return early at `tokenBudget.ts:51-53`). `checkTokenBudget` returns either:

- `continue` — inject a user nudge message and loop. Thresholds: `< 90% of budget` and not "diminishing" (defined as `>=3 continuations and last-delta/current-delta both <500 tokens`, `tokenBudget.ts:59-62`).
- `stop` — with a `completionEvent` if any continuation happened, logged as `tengu_token_budget_completed`.

### 6.5. Abort paths

Two abort checks, both after streaming (`src/query.ts:1015-1052`) and after tool execution (`src/query.ts:1485-1516`). Streaming-mode executor's `getRemainingResults()` is drained even on abort to generate synthetic tool_result blocks for in-flight tools — otherwise the tool_use without tool_result would leave an invalid transcript. Chicago-MCP cleanup fires here too. Interruption messages are skipped if `signal.reason === 'interrupt'` because the queued user message that follows provides the context.

### 6.6. maxTurns and hook-stopped continuation

`maxTurns` (if set in params) is checked twice:
- At the top of tool-abort handling (`src/query.ts:1507-1514`) — yields a `max_turns_reached` attachment before returning.
- Before the recursive state build (`src/query.ts:1704-1712`) — same but normal path.

`shouldPreventContinuation` (`src/query.ts:1388-1392`, `1519-1521`) fires when a PostToolUse hook emits a `hook_stopped_continuation` attachment. Terminal: `{ reason: 'hook_stopped' }`.

---

## 7. "Decide what to do next" — the decision table

Summarizing the 10+ branches the loop can take at the end of each iteration:

| Condition                                                        | Action                                        | File:Line                      |
|------------------------------------------------------------------|-----------------------------------------------|--------------------------------|
| Abort signal + streaming                                         | Return `aborted_streaming`                    | `query.ts:1015-1052`           |
| `needsFollowUp === false` and withheld 413                       | Try collapse-drain → reactive-compact → error | `query.ts:1070-1183`           |
| `needsFollowUp === false` and withheld media-size                | Reactive-compact                              | `query.ts:1082-1175`           |
| Withheld max_output_tokens, capEnabled                           | Escalate to 64k and continue                  | `query.ts:1195-1221`           |
| Withheld max_output_tokens, recovery < 3                         | Inject "Resume directly" and continue         | `query.ts:1223-1252`           |
| Last message is API error                                        | Run `executeStopFailureHooks` + return        | `query.ts:1262-1265`           |
| Stop hook `preventContinuation`                                  | Return `stop_hook_prevented`                  | `query.ts:1278-1280`           |
| Stop hook blocking errors                                        | Continue with errors appended                 | `query.ts:1282-1306`           |
| Token-budget says continue                                       | Inject nudge + continue                       | `query.ts:1316-1341`           |
| All stop-hook gates pass                                         | Return `completed`                            | `query.ts:1357`                |
| `shouldPreventContinuation` (PostToolUse hook)                   | Return `hook_stopped`                         | `query.ts:1519-1521`           |
| Abort signal during tool execution                               | Return `aborted_tools`                        | `query.ts:1485-1516`           |
| `maxTurns` exceeded                                              | Return `max_turns`                            | `query.ts:1705-1712`           |
| Normal tool-result path                                          | Rebuild `State`, `turnCount++`, continue      | `query.ts:1715-1727`           |

---

## 8. Key data structures

**`ToolUseBlock`** — Anthropic SDK type, the atomic "act" signal. Content: `{ id, name, input }`.

**`State` (`query.ts:204-217`)** — 10 fields, all rebuilt at `continue` sites. Notable fields:

- `transition: Continue | undefined` — *why* the previous iteration continued. Kept on the State specifically to let tests assert recovery paths fired without inspecting message contents (`query.ts:215-216`). Also load-bearing: the 413 collapse-drain path gates on `state.transition?.reason !== 'collapse_drain_retry'` (`query.ts:1092`) to prevent infinite re-draining.
- `hasAttemptedReactiveCompact` — one-shot guard. Preserved across stop-hook-blocking retries (not reset to false) to break the compact/blocking death-spiral (`query.ts:1292-1297`).
- `pendingToolUseSummary` — `Promise<ToolUseSummaryMessage | null>`. Haiku runs in parallel (~1s) with main-model streaming (5-30s); awaited on the *next* iteration (`query.ts:1055-1060`) so it never blocks the critical path.
- `maxOutputTokensRecoveryCount` vs `maxOutputTokensOverride` — the former counts toward `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`, the latter is the one-shot 8k→64k escalation slot.
- `stopHookActive` — tells `handleStopHooks` that the *previous* iteration already dealt with blocking errors, so hooks that "re-fire on unchanged state" are suppressed.

**`QueryConfig` (`query/config.ts`)** — immutable per-`query()` snapshot: `sessionId`, and four gates (`streamingToolExecution`, `emitToolUseSummaries`, `isAnt`, `fastModeEnabled`). Explicitly excludes `feature()` gates — those must stay inline for dead-code elimination at the bundle stage (`config.ts:13-14`).

**`QueryDeps` (`query/deps.ts`)** — 4 injection seams for tests: `callModel`, `microcompact`, `autocompact`, `uuid`. Production factory at `deps.ts:33-40`.

**`QueryChainTracking`** — `{ chainId, depth }` in `toolUseContext.queryTracking`. Logged on every analytics event. Depth increments per query chain (initialized at `query.ts:347-355` and incremented from inherited context). Enables cross-subagent tracing.

**`Continue` / `Terminal` types** — imported from `./query/transitions.js` (I couldn't locate the file — it's either dynamically generated or in a build-time-only location). The concrete shape is inferrable from the `{ reason: 'completed' | 'aborted_streaming' | ... }` return sites: Terminal is a tagged union of terminal reasons; Continue is a tagged union of continuation reasons (`collapse_drain_retry`, `reactive_compact_retry`, `max_output_tokens_escalate`, `max_output_tokens_recovery`, `stop_hook_blocking`, `token_budget_continuation`, `next_turn`). **Follow-up needed** — the file path exists in the import but I couldn't resolve the source file. Worth a dedicated grep.

---

## 9. Control flow diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│  query(params) → queryLoop(params, consumedCommandUuids)            │
│    state = initialState; config = buildQueryConfig()                │
│    pendingMemoryPrefetch = startRelevantMemoryPrefetch()            │
└──────────────────┬──────────────────────────────────────────────────┘
                   │
           ┌───────▼────────┐  ── iteration top
           │ destructure    │
           │ state → {msgs, │
           │ toolUseCtx,... │
           └───────┬────────┘
                   │
          ┌────────▼─────────┐
          │ PREPARE          │  snip → microcompact → collapse →
          │ CONTEXT          │  autocompact → (optional blocking limit)
          └────────┬─────────┘
                   │
          ┌────────▼─────────┐
          │ THINK (stream)   │  deps.callModel(...)
          │  toolUseBlocks[] │    yields StreamEvent | Message
          │  needsFollowUp   │    withholds 413/media/maxTokens errors
          │  streamingExec.  │    streamingTools start mid-stream
          │    addTool()     │
          └────────┬─────────┘
                   │
                   ▼
              needsFollowUp?
                 /      \
              no        yes
               │         │
        ┌──────▼────┐    │
        │ RECOVERY  │    │    ┌────────────────────────┐
        │  -collapse│    └───▶│ ACT                    │
        │  -reactive│         │  runTools / streamingE │
        │  -maxTok  │         │   partition concurrent │
        │  -stopHook│         │   runToolUse per block │
        │  -budget  │         │    preHooks → perm →   │
        └──────┬────┘         │    call → postHooks    │
               │              └──────────┬─────────────┘
               │                         │
               │                  ┌──────▼─────────┐
               │                  │ OBSERVE/MERGE  │
               │                  │ +queue atts    │
               │                  │ +memory att    │
               │                  │ +skill att     │
               │                  │ +file changes  │
               │                  └──────┬─────────┘
               │                         │
               │                 maxTurns reached? ─── yes ─┐
               │                         │ no               │
               ▼                         ▼                  ▼
          RETURN{reason}             state = next       RETURN{max_turns}
                                     turnCount++
                                         │
                                         └─► iteration top
```

---

## 10. Integration points with other subsystems

1. **Anthropic SDK** — via `queryModelWithStreaming` (`services/api/claude.js`), wrapped by `deps.callModel`. Handles streaming, tool protocol, fallback model (`FallbackTriggeredError`), thinking-block signatures, cache control, and the 2026-03-13 task_budget beta (`src/query.ts:191-197`, `src/query.ts:697-706`).

2. **Compaction subsystems** — three competing/composing mechanisms:
   - `snipCompact` (HISTORY_SNIP) — removes oldest tool outputs, leaves structure.
   - `microcompact` (always on) — cached-output deletion.
   - `contextCollapse` (CONTEXT_COLLAPSE) — lazy collapse commit log.
   - `autoCompact` (always on) — full summary.
   - `reactiveCompact` (REACTIVE_COMPACT) — on-413 recovery.
   The ordering comments are worth preserving: see `src/query.ts:396-410`, `430-447`. **Follow-up worth it** — the compaction stack deserves a dedicated deep-dive.

3. **Stop-hooks subsystem** — `utils/hooks.js` and `query/stopHooks.ts`. Includes REPLHookContext, the hook-interleaved message format, teammate-idle/task-completed hooks. The teammate path is the integration with the team coordination system (`getTaskListId`, `listTasks`, `isTeammate`). **Deserves dedicated analysis.**

4. **Tool registration** — `options.tools` on `ToolUseContext`. Consumed via `findToolByName`. Refreshed between turns via `options.refreshTools()` to pick up newly-connected MCP servers (`src/query.ts:1660-1671`).

5. **Permission system** — `CanUseToolFn` (`hooks/useCanUseTool.ts`), `resolveHookPermissionDecision`, transcript auto-mode classifier, bash speculative classifier. The integration between hooks and permissions is tangled (`toolExecution.ts:921-1103`) — hooks can override permissions ("PermissionRequest" hook), update input, prevent continuation, etc.

6. **Subagent / teammate system** — `toolUseContext.agentId` is the main switch. Gates template-job classification, background bookkeeping skips, queue scoping, task-completed hooks, CU cleanup (Chicago-MCP).

7. **Analytics / OTel** — `logEvent` (Tengu) and `logOTelEvent` are called densely throughout (literally dozens of sites). `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` branded type forces explicit acknowledgement that the value is not PII before it can be logged.

8. **Command queue** — `messageQueueManager` (`utils/messageQueueManager.ts`). Queue drain rules are agent-scoped (`src/query.ts:1565-1578`) and live priority-based (`getCommandsByMaxPriority('next' | 'later')`).

9. **Prefetch pipeline** — three prefetches fire in parallel during streaming: memory (`startRelevantMemoryPrefetch`, turn-scoped), skill discovery (`skillPrefetch.startSkillDiscoveryPrefetch`, per-iteration), tool-use summary (Haiku, per-iteration). Memory uses a `using` dispose statement (`src/query.ts:301`) which is the critical cleanup hook — see MemoryPrefetch for dispose/telemetry semantics.

10. **Bridge / template jobs** — `CLAUDE_JOB_DIR` env and `jobs/classifier.js` (TEMPLATES feature). The stop-hook awaits classification (60-s budget) so `claude list` never shows stale state.

---

## 11. Notable comments and historical incidents captured in code

- **"The rules of thinking"** (`src/query.ts:151-162`) — protected-thinking blocks are model-bound, must be preserved for an assistant trajectory's duration, must never be the last block in a message. Violating these costs "an entire day of debugging and hair pulling."
- **The compact/blocking death spiral** (`src/query.ts:1292-1297`) — do not reset `hasAttemptedReactiveCompact` on stop-hook blocking retry. Prior incident: "infinite loop: compact → still too long → error → stop hook blocking → compact → … burning thousands of API calls."
- **Phantom user interrupts on Node 18** (`src/query.ts:985-989`) — surfacing the real model/runtime error instead of "Request interrupted by user" was in response to SDK consumers seeing phantom interrupts on Node 18's missing `Array.prototype.with()`.
- **VCR fixture hash stability** (`src/query.ts:766-773`, `toolExecution.ts:1181-1205`) — backfilled inputs must not reach `tool.call()`, because tool result strings embed the input path verbatim and any change breaks transcript hashes.
- **Chicago-MCP cleanup** (`src/query.ts:1033-1042`, `stopHooks.ts:159-173`) — the CU lock is a process-wide module-level variable; subagent-released locks leak main-thread cleanup and unhide mid-turn. Only fires on `!agentId`.

---

## 12. Recommendations for follow-up analysis

1. **Compaction stack deep-dive** — snip vs. microcompact vs. collapse vs. autocompact vs. reactive-compact is a five-way coexistence puzzle. The ordering comments hint at several past incidents that aren't documented in git history.
2. **`query/transitions.ts`** — the `Continue` and `Terminal` types import from a file I couldn't resolve. Either it's generated at build-time or lives outside the src tree. Worth locating and cataloguing all continuation/terminal reasons in a single reference table.
3. **Stop-hooks subsystem** — `utils/hooks.js`, the REPLHookContext, teammate hooks, and the `executeStopFailureHooks` / `executePostSamplingHooks` / `executePermissionDeniedHooks` zoo all deserve a dedicated doc. They form the "extensibility" surface of the agent loop.
4. **`ToolUseContext`** — 15+ fields referenced throughout; a dedicated reference that covers `queryTracking`, `options.tools`, `options.mcpClients`, `getAppState`, `toolDecisions`, `abortController`, `setInProgressToolUseIDs`, `agentId`, etc. The contextModifier pattern (tools returning functions to update the context) is subtle.
5. **StreamingToolExecutor internals** — the `TrackedTool` state machine, progress-signaling, `siblingAbortController` semantics, and the discard/rebuild on fallback deserve their own doc. The integration between mid-stream `addTool` and post-stream `getRemainingResults` is the trickiest piece.
6. **Permission + hook interleaving** — the `hookPermissionResult / hookUpdatedInput / shouldPreventContinuation / stopReason` quadruple that `runPreToolUseHooks` can yield is load-bearing but scattered. A state-machine diagram would help.
7. **Queue routing under agent scoping** — `isSlashCommand`, `cmd.mode === 'task-notification'`, `cmd.agentId === undefined` — the current doc cites two rationales (main-thread-only prompts, subagent-only task-notifications) but doesn't cover `agent:*` querySources or `runForkedAgent` contributors.
8. **Token budget vs. taskBudget beta** — the loop carries both `budgetTracker` (custom client-side budget) and `taskBudget` (server-side Anthropic beta, `params.taskBudget`). They're distinct and the comment at `src/query.ts:193-197` is the only documentation — worth a side-by-side.
