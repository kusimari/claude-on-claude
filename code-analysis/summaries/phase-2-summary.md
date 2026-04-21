# Phase 2 Summary — REPL, Query Loop, Prompt, API, Parsing, Agent Loop

Covers reports 06–11: interactive UI input, query orchestration, prompt construction, Claude API communication, response parsing, inner agent loop.

---

## 06 — REPL UI Input

**Complexity**: Very Complex

### Key findings
- `REPL` component (src/screens/REPL.tsx:324, ~5000 LOC) is the interactive TUI root — renders PromptInput, Logo, Logs, MessageHistory, dialogs; consumes `useAppState` for message list, tool state, queue, mode, dialogs.
- **QueryGuard** state machine (src/screens/REPL.tsx:897-900) is the single serialization point for every query: `idle → queued → running → completing → idle`, enforces one-query-at-a-time invariant; queued second prompts resolve via `pendingQueueRef`.
- `onQuery()` path: text → `processUserInput()` → slash-command extraction → `query()` generator → drains via `for await` into `useAppState.messages` + transcript writer; aborts via `abortController.abort('interrupt' | 'escape')`.
- Slash commands split pre-loop (locally resolved) vs in-loop (need model); `processSlashCommand` dispatches to skill/command registry; `SlashCommand` attachments vs text input differ in tool routing.
- Input modes: normal, paste-buffer (bracketed paste multi-line), command-palette (Ctrl+K), image-paste, history-cycle — all mediated by `PromptInput` with `useInputMode` hook.

### Critical code locations
- `src/screens/REPL.tsx:324` main component; `:897-900` QueryGuard; `:1156-1289` `onQuery`; `:1601-1678` stream consumer; `:2189-2240` abort path; `:3014-3098` paste handling.
- `src/components/PromptInput.tsx` — input capture, history, image paste, bracketed paste.
- `src/hooks/useAppState.ts` — Zustand-like selector pattern, `setAppState()` mutates slice + triggers re-render.
- `src/hooks/useInputMode.ts` — mode dispatch.

### Integration points
- **query.ts**: consumer of `query()` async generator; yields `Message` + stream events; state stitching via `useAppState.setMessages`.
- **Hooks system**: `SessionStart` fires at REPL mount; `UserPromptSubmit` hooks run pre-query.
- **Transcript**: `writeTranscript()` serializes messages on 100ms flush interval; direct-mutation invariant on `AssistantMessage.message` (see report 10).
- **Command queue**: `messageQueueManager` drained inside loop — slash commands excluded (reprocessed through `processSlashCommand`).

### Recommendations
- Deep dives: PromptInput (paste/image/history state), useAppState selector perf, dialog orchestration (10+ modal types), slash-command registry.
- REPL.tsx is ~5000 LOC — follow-up would benefit from a sub-component breakdown map.

---

## 07 — Query Orchestration (query.ts)

**Complexity**: Very Complex

### Key findings
- `query()` (src/query.ts:219-239) is a thin wrapper around `queryLoop()` (src/query.ts:241-1728); only special behavior is `notifyCommandLifecycle('completed')` on clean return (skipped on throw/`.return()`).
- Two stacked async generators: outer loop yields per-turn `Terminal` reason; inner `deps.callModel()` yields stream events + assembled `AssistantMessage`s.
- **7 continuation reasons** (`Continue`): `collapse_drain_retry`, `reactive_compact_retry`, `max_output_tokens_escalate`, `max_output_tokens_recovery`, `stop_hook_blocking`, `token_budget_continuation`, `next_turn`.
- **12 Terminal exit reasons**: `completed`, `aborted_streaming`, `aborted_tools`, `blocking_limit`, `image_error`, `model_error`, `prompt_too_long`, `stop_hook_prevented`, `hook_stopped`, `max_turns`, etc.
- `State` (src/query.ts:204-217, 10 fields) rebuilt at each `continue` site — only `toolUseContext` mutates mid-iteration; `transition` field is load-bearing for gating recovery paths (e.g. collapse-drain one-shot).
- **Withheld errors**: 413/media/max_output_tokens errors surfaced during streaming are held back so SDK callers don't see recoverable intermediates — replayed only after `needsFollowUp === false` check.

### Critical code locations
- `src/query.ts:219-239` `query()`; `:241-1728` `queryLoop()`; `:204-217` `State`; `:151-162` "rules of thinking" comment.
- `:361-548` pre-API context pipeline; `:552-557` `needsFollowUp` invariant comment; `:659-708` streaming consumption; `:712-740` streaming fallback tombstoning.
- `:1062-1358` stop condition branches; `:1089-1117` collapse-drain; `:1119-1175` reactive-compact; `:1199-1252` max-output-tokens recovery; `:1267-1306` stop-hook handling; `:1292-1297` compact/blocking death-spiral guard comment.
- `src/query/stopHooks.ts:65-472` `handleStopHooks`; `:335-452` teammate `TaskCompleted`/`TeammateIdle`.
- `src/query/deps.ts`, `src/query/config.ts` — 4-seam injection (`callModel`, `microcompact`, `autocompact`, `uuid`) + immutable per-session `QueryConfig`.

### Integration points
- **Compaction stack**: 5-way interaction — snip, microcompact, contextCollapse, autoCompact, reactiveCompact — with strict ordering (`snip before microcompact`, `collapse before autocompact`, `reactive owns 413`).
- **Stop hooks**: `executeStopHooks`, `executeStopFailureHooks`, `TaskCompleted` (teammate), `TeammateIdle` — can `preventContinuation` or `blockingErrors` for retry.
- **Token budget**: `checkTokenBudget` (gated `TOKEN_BUDGET`) injects continuation nudges; skips subagents; diminishing-returns guard.
- **Prefetches** (parallel with streaming): `startRelevantMemoryPrefetch` (turn-scoped, `using` dispose), skill discovery, Haiku tool-use summary.
- **Chain tracking**: `QueryChainTracking` (`chainId`, `depth`) on `toolUseContext.queryTracking` — logged to every analytics event for subagent tracing.

### Recommendations
- Dedicated compaction-stack doc (5 interacting mechanisms).
- Locate `src/query/transitions.ts` — `Continue`/`Terminal` types imported but file not found in src tree.
- Stop-hook subsystem warrants its own deep dive (teammate hooks, REPLHookContext, `executePostSamplingHooks` zoo).

### Invariants
- `stop_reason === 'tool_use'` not used — content-block presence (`needsFollowUp`) is authoritative.
- `hasAttemptedReactiveCompact` preserved across stop-hook-blocking retries — resetting causes death-spiral.
- Protected-thinking blocks model-bound; stripped before fallback on ant builds.
- `collapse_drain_retry` transition is one-shot (checked via `state.transition?.reason`).

---

## 08 — Prompt Construction

**Complexity**: Complex

### Key findings
- `getSystemPrompt()` (src/constants/prompts.ts:444-577) assembles system prompt in two regions separated by `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` (prompts.ts:114) — a byte-exact marker that splits prefix (cache-stable) from suffix (per-session).
- `splitSysPromptPrefix()` (src/utils/api.ts:321-435) partitions system content blocks: prefix gets `cache_control: { type: 'ephemeral', ttl: '1h' | '5m' }`; suffix left uncached.
- **Prompt cache discipline**: any whitespace/byte change in prefix invalidates all downstream cached hits. Beta header latches (`afkModeHeaderLatched`, `promptCache1hAllowlistLatched`) are sticky-on per session, cleared by `/clear` + `/compact` to synchronize cache boundaries.
- User context prepended byte-stably: `prependUserContext()` in query.ts:660 injects directory listing, git status, recent file-changes, memory — all before turn messages, after system prompt.
- Model-specific sections (thinking config, max_tokens, beta headers) are gated by `isClaudeX()` helpers; some features beta-gated (`task_budget`, `token_efficient_tools`).

### Critical code locations
- `src/constants/prompts.ts:444-577` `getSystemPrompt`; `:114` `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`; tool-descriptions, CLAUDE.md, agent-descriptions injected pre-boundary.
- `src/utils/api.ts:321-435` `splitSysPromptPrefix`; ttl selection logic.
- `src/query.ts:660` `prependUserContext` call site.
- `src/utils/userContext.ts` — directory + git + file-changes assembly.
- `src/services/api/claude.ts:1596-1629` thinking-config request-side setup.

### Integration points
- **Memory**: CLAUDE.md files (project, user, enterprise) read at `startRelevantMemoryPrefetch` and injected into userContext.
- **Tools**: tool descriptions rendered in system prompt prefix — pool ordering (see report 03) is cache-load-bearing.
- **Agents**: agent definitions (Task tool subagents) injected pre-boundary.
- **Settings**: `beta` header list computed from enabled features; latches prevent mid-session flip.

### Recommendations
- Document the full set of beta-header latches + their clear triggers.
- Cache-ttl decision tree deserves a reference (5m vs 1h selection).
- Test coverage on boundary byte-stability across model switches.

### Invariants
- System-prompt prefix ordering fixed; any reorder busts cache.
- One `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker per prompt (invariant, not enforced in code).
- Beta-header latches sticky-on through session; only `/clear` + `/compact` clear them.

---

## 09 — API Interaction

**Complexity**: Very Complex

### Key findings
- `getAnthropicClient()` (src/services/api/client.ts) provider abstraction: firstParty, Bedrock, Vertex, CCR, grove; handles base URL, header injection, auth source, region-specific behavior.
- `queryModel()` (src/services/api/claude.ts:1017, within 3419 LOC file) is the streaming async generator — opens raw stream via `.withResponse()` (not `BetaMessageStream`, to avoid O(n²) `partialParse` on tool inputs).
- `withRetry.ts` (822 LOC) — exponential backoff with jitter, respects `Retry-After`, distinguishes retryable (5xx, 408, 429, connection-timeout) from non-retryable (4xx, refusal, context-window). `FallbackTriggeredError` (model-under-load) bubbles to outer query loop.
- Idle watchdog: `STREAM_IDLE_TIMEOUT_MS = 90s` (env-overridable) aborts silent connections; `STALL_THRESHOLD_MS = 30s` emits `tengu_streaming_stall` telemetry; both separate from SDK's request timeout (which only covers initial fetch).
- Non-streaming fallback: on stream failure (no events / incomplete), calls `executeNonStreamingRequest()` (claude.ts:818-905) with per-attempt timeout 120s (remote) or 300s (local); escape hatch `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` exists to avoid double-execution incidents (inc-4258).

### Critical code locations
- `src/services/api/client.ts` — `getAnthropicClient`, provider dispatch, header injection.
- `src/services/api/claude.ts:752-780` `queryModelWithStreaming` VCR wrapper; `:1017` `queryModel`; `:1818-1836` raw stream create; `:1868-1929` idle watchdog; `:2404-2569` non-streaming fallback; `:2612-2646` 404 stream-creation fallback; `:2434-2462` abort-vs-timeout distinction.
- `src/services/api/withRetry.ts` — retry loop, backoff, Retry-After, `FallbackTriggeredError`.
- `src/services/api/errors.ts:1184-1207` `getErrorMessageIfRefusal`.
- `src/services/vcr.ts:88-170` `withStreamingVCR` — test fixture record/replay.
- `src/remote/sdkMessageAdapter.ts:45-218` — SDK daemon ↔ stream-event translation for remote sessions.

### Integration points
- **VCR**: records live streams, dehydrates file-contents for reproducibility, adds cached cost back to session tracker.
- **Provider-specific**: `normalizeModelStringForAPI` (claude.ts:867) rewrites model names; `CLIENT_REQUEST_ID_HEADER` firstParty-only; CCR coalesces via `ccrClient.ts:86-184`.
- **Cost tracking**: `addToTotalSessionCost` per message_delta; `cost-tracker.ts:271` reads `server_tool_use.web_search_requests`.
- **Transcript**: `request_id`, `stream_response` captured for replay/debug.

### Recommendations
- Dedicated deep dive on `withRetry.ts` (822 LOC) — backoff policy, Retry-After parsing, jitter.
- Document provider-abstraction seams for future model provider integrations.
- `tengu_streaming_stall` threshold tuning data would inform watchdog constants.

### Invariants
- Raw stream (not `BetaMessageStream`) — O(n²) partial-parse bypass.
- Idle watchdog distinct from SDK request timeout.
- Abort-vs-timeout: `signal.aborted === true` → real user ESC; false → rethrow as `APIConnectionTimeoutError` for retry.

---

## 10 — API Response Handling and Parsing

**Complexity**: Very Complex

### Key findings
- Per-block streaming accumulator in `queryModel()` — yields `{type: 'assistant', message}` **once per content block** at `content_block_stop` (not at message end); yields raw `stream_event` passthrough every iteration for UI + transport consumers.
- **Direct mutation invariant** (claude.ts:2236-2248): `lastMsg.message.usage = usage` (not replacement) — transcript write queue holds reference, serializes on 100ms flush; object replacement decouples queued ref. Load-bearing.
- `normalizeContentFromAPI()` (src/utils/messages.ts:2651-2751): `tool_use` inputs parsed with `safeParseJSON` + `{}` fallback; `text` preserved byte-exact for cache; `thinking` passthrough preserves signature. Known hole: recursive stringified JSON (messages.ts:2670-2674).
- `StreamingToolExecutor` (src/services/tools/StreamingToolExecutor.ts, 530 LOC) dispatches tools mid-stream: concurrent-safe run in parallel, non-safe block queue for exclusive access; `siblingAbortController` cascades Bash errors only (Read/WebFetch independent).
- Stop-reason mapping: `end_turn` → exit; `tool_use` → not read (content-presence authoritative); `max_tokens` + `model_context_window_exceeded` → reuse max-output-tokens recovery; `refusal` → synthesized AUP message via `getErrorMessageIfRefusal`.

### Critical code locations
- `src/services/api/claude.ts:1940-2304` event switch; `:1980-1993` message_start; `:1995-2051` content_block_start per-type init; `:2053-2169` content_block_delta; `:2171-2211` content_block_stop; `:2213-2293` message_delta.
- `:2236-2248` direct-mutation invariant comment; `:2924-2987` `updateUsage`; `:2993-3038` `accumulateUsage`.
- `src/utils/messages.ts:2651-2751` `normalizeContentFromAPI`; `:2661-2719` tool_use; `:2721-2730` text byte-preservation.
- `src/services/tools/StreamingToolExecutor.ts:40-205` state machine + synthetic errors; `:265-405` executeTool; `:412+` getCompletedResults.
- `src/query.ts:826-862` consumer loop with `needsFollowUp`; `:712-740` streaming fallback tombstoning.

### Integration points
- **Tool execution**: `StreamingToolExecutor.addTool()` mid-stream → `executeTool()` → `tool.call()` → yielded `MessageUpdate` stream → `getCompletedResults()` drained into `toolResults[]`.
- **Cost tracking**: `updateUsage` non-null-and-nonzero guard (message_start output_tokens:0 guarded); `accumulateUsage` session-level; `cache_deleted_input_tokens` off-type for DCE.
- **Prompt cache**: `checkResponseForCacheBreak` (gated `PROMPT_CACHE_BREAK_DETECTION`); quota status extracted from response headers.
- **Remote / VCR / CCR**: three adapters normalize `BetaRawMessageStreamEvent` in/out — must stay in sync.

### Recommendations
- Flagged as intricate in deliverable. Follow-ups: normalization+per-tool-input audit; executor discard/rebuild + streaming fallback dance; idle/stall/abort timer interaction; stop-reason recovery state-machine diagram; usage-merge policy reference.
- Comments reference inc-3694, inc-4029, inc-4258, gh-#21056, gh-#1513 — all post-incident invariants.

### Invariants
- One `AssistantMessage` per content block; consumer stitches blocks.
- Direct mutation (not replacement) on `lastMsg.message` for transcript flush.
- `text` content byte-exact; any whitespace edit breaks cache.
- Thinking signatures model-bound; stripped before fallback; tombstoned on streaming fallback.

---

## 11 — Inner Agent Loop

**Complexity**: Very Complex

### Key findings
- Think-Act-Observe implemented in `queryLoop()` generator (src/query.ts:241-1728); single loop body per `while (true)` iteration.
- **Single exit signal**: `needsFollowUp` flipped true when any `tool_use` block arrives in any streamed assistant message (query.ts:829-835). `stop_reason` not read — known-unreliable.
- **Pre-API pipeline** (strict order, query.ts:365-548): `getMessagesAfterCompactBoundary` → `applyToolResultBudget` → `snipCompactIfNeeded` → `microcompact` → `contextCollapse.applyCollapsesIfNeeded` → `autocompact` → hard `blocking_limit` check. Order rationale in comments (query.ts:396-399).
- **Two execution backends**: `StreamingToolExecutor` (gated `streamingToolExecution`) runs tools mid-stream with concurrency safety; else `runTools` (toolOrchestration.ts:19-82) partitions into concurrent batches + serial singletons.
- **10+ decision branches** at iteration end: collapse-drain retry, reactive-compact retry, max-tokens escalate/resume, stop-hook preventContinuation/blocking, token-budget nudge, abort paths, maxTurns, hook-stopped, normal next-turn.

### Critical code locations
- `src/query.ts:204-217` `State` type; `:241-1728` `queryLoop`; `:361-548` pre-API; `:659-708` streaming; `:829-862` consumer + executor dispatch.
- `:1015-1052`, `:1485-1516` abort paths; `:1062-1358` stop-condition branches; `:1704-1712` maxTurns; `:1715-1727` next-turn state rebuild.
- `src/services/tools/toolOrchestration.ts:19-82` `runTools`; `:91-116` `partitionToolCalls`; `:152-177` `runToolsConcurrently`.
- `src/services/tools/toolExecution.ts:337-490` `runToolUse` per-tool state machine; `:492-570` `streamedCheckPermissionsAndCallTool`; `:800-862` `runPreToolUseHooks`; `:921-1103` permission resolution + `PermissionDenied` hooks; `:1540-1542` MCP post-hook ordering.
- `src/services/tools/StreamingToolExecutor.ts` TrackedTool state machine, siblingAbortController.
- `src/query/stopHooks.ts:65-472` `handleStopHooks`; `src/query/tokenBudget.ts` continuation decision.

### Integration points
- **Permission + hook interleaving** (toolExecution.ts:921-1103): hooks can override permissions, update input, stop tool, retry on denial; tangled state quadruple (`hookPermissionResult`, `hookUpdatedInput`, `shouldPreventContinuation`, `stopReason`).
- **MCP tool ordering**: post-hook runs before `addToolResult(toolOutput)` because hooks rewrite output; non-MCP tools get pre-mapped result first.
- **Subagent/teammate**: `agentId` is main switch — gates template-job classification, background bookkeeping, queue scoping, `TaskCompleted` hooks, CU (Chicago-MCP) cleanup.
- **Concurrent context modifiers**: concurrent-safe tools queue `modifyContext` callbacks per tool-use-id, apply post-batch; serial tools apply immediately.
- **Prefetch pipeline**: memory (`using` dispose), skill discovery, Haiku tool-use summary — all parallel to streaming, results awaited non-blockingly next iteration.

### Recommendations
- Compaction-stack deep-dive (5 interacting mechanisms with historical incidents).
- Stop-hooks subsystem (teammate hooks, REPLHookContext, execute*Hooks zoo).
- `StreamingToolExecutor` internals (TrackedTool, sibling aborts, discard/rebuild).
- Permission + hook state-machine diagram (load-bearing but scattered).
- `src/query/transitions.ts` — source file unresolved (generated or outside src); catalog all `Continue`/`Terminal` reasons.

### Invariants (from code comments)
- **"Rules of thinking"** (query.ts:151-162): protected-thinking blocks model-bound, preserved for trajectory duration, never the last block.
- **Compact/blocking death-spiral guard** (query.ts:1292-1297): `hasAttemptedReactiveCompact` preserved across stop-hook-blocking retries.
- **Phantom interrupt** (query.ts:985-989): distinguish real abort from Node 18 `Array.prototype.with()` failures.
- **VCR fixture hash stability** (query.ts:766-773, toolExecution.ts:1181-1205): backfilled inputs must not reach `tool.call()`.
- **Chicago-MCP cleanup**: only on `!agentId` (main thread); subagent-released locks leak.

---

## Cross-cutting themes (Phase 2)

- **Content-presence, not stop_reason**: the loop's authoritative continue/stop signal is whether tool_use blocks appeared — `stop_reason === 'tool_use'` is documented-unreliable.
- **Prompt cache discipline pervasive**: byte-exact text preservation, system-prompt boundary marker, sticky beta-header latches, built-in tool prefix ordering — all preserve server-side cache keys across turns.
- **Per-block streaming**: `AssistantMessage`s yielded once per completed content block (not per message); consumer stitches; tools dispatched mid-stream.
- **Withheld errors**: 413/media/max_output_tokens held during streaming, replayed only after `needsFollowUp` check, so SDK callers don't fail on recoverable intermediates.
- **5-way compaction stack**: snip → microcompact → collapse → autocompact → reactiveCompact, each with ordering invariants and historical-incident justifications.
- **Direct mutation over replacement**: load-bearing for transcript flush queue (usage/stop_reason), `State` rebuild at `continue` sites.
- **Prefetch parallelism**: memory, skill discovery, Haiku summary all run parallel to streaming/tools, drained next iteration; ~98% hidden behind main turn.
- **Tombstoning on fallback**: partial-stream assistant messages, in-flight tool results, thinking signatures all invalidated + reset on streaming/model fallback to avoid replay-of-stale-state 400s.
- **Permission + hook tangled**: hooks can override permissions, rewrite input, prevent continuation; MCP tool ordering differs because hooks rewrite output.
- **Injection seams**: 4 `QueryDeps` (callModel, microcompact, autocompact, uuid); `QueryConfig` immutable per-session; tests get injection without spy-per-module.
