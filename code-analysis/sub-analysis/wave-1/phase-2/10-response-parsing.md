# 10 — API Response Handling and Parsing

**Scope**: How the stream of SSE events from Anthropic's `beta.messages` endpoint
becomes the assistant messages and tool actions the rest of the CLI acts on.
The primary surface area is **`src/services/api/claude.ts`** (3419 LOC) — in
particular the accumulator inside `queryModel()` (the streaming async
generator) at lines 1017–2900+ — plus `normalizeContentFromAPI()` in
**`src/utils/messages.ts:2651`** and the consumer loop in **`src/query.ts:659`**.
Tool execution fan-out is handled by `StreamingToolExecutor`
(`src/services/tools/StreamingToolExecutor.ts`, 530 LOC).

This pipeline is **intricate**. The deliverable's flag-if-complex criterion
applies; §14 enumerates the specific state-machine, fallback, and mutation
subtleties that deserve dedicated follow-up.

---

## 1. Entry points and pipeline shape

The function that `query.ts` actually calls via `deps.callModel(...)` is
`queryModelWithStreaming()` (`claude.ts:752`), a thin wrapper that delegates to
the VCR recording/playback layer before yielding to the streaming generator:

```ts
// claude.ts:752-780
export async function* queryModelWithStreaming({...}): AsyncGenerator<
  StreamEvent | AssistantMessage | SystemAPIErrorMessage, void
> {
  return yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options)
  })
}
```

The three output shapes that flow out of this generator are:

- `{ type: 'stream_event', event: BetaRawMessageStreamEvent, ttftMs? }` — raw
  passthrough of each SSE event (claude.ts:2299–2303); consumed by the UI,
  remote transports, and a few tools (notably `WebSearchTool`).
- `{ type: 'assistant', message: BetaMessage, uuid, timestamp, ... }` —
  assembled `AssistantMessage` yielded **once per content block**, at
  `content_block_stop` (claude.ts:2192–2211). Also yielded by the non-streaming
  fallback (claude.ts:2571–2594).
- `SystemAPIErrorMessage` / refusal / max_tokens / context-window error —
  surfaced from stop-reason handling (claude.ts:2258–2291) or from the retry
  generator's intermediate yields.

Everything from here on lives inside `queryModel()`.

---

## 2. Raw-stream construction and idle watchdog

### 2.1 Why raw stream, not `BetaMessageStream`

The stream is opened with `.withResponse()` on the SDK's raw POST, not the
higher-level `BetaMessageStream` helper (claude.ts:1818–1836). The reason,
called out in the comment: `BetaMessageStream` calls `partialParse()` on every
`input_json_delta`, making tool-input accumulation O(n²) where n is the number
of deltas. The CLI instead appends deltas to a string and only parses once at
`content_block_stop`. This is a deliberate performance choice.

```ts
// claude.ts:1822-1832
const result = await anthropic.beta.messages
  .create({ ...params, stream: true }, { signal, ...headers })
  .withResponse()
streamRequestId = result.request_id
streamResponse = result.response
return result.data  // raw Stream<BetaRawMessageStreamEvent>
```

### 2.2 Watchdog + stall detection

`claude.ts:1868–1929` installs a dual-timer idle watchdog:

- `STREAM_IDLE_WARNING_MS = STREAM_IDLE_TIMEOUT_MS / 2` (default 45s): debug log
  only.
- `STREAM_IDLE_TIMEOUT_MS = 90_000` (overridable via
  `CLAUDE_STREAM_IDLE_TIMEOUT_MS`): fires `releaseStreamResources()` which
  cancels the underlying response body and aborts the fetch.

Both timers are reset on every chunk (`resetStreamIdleTimer()`,
claude.ts:1941). **This is separate from the SDK's request timeout**, which
only covers the initial fetch, not the streaming body. Without this watchdog a
silently dropped connection can hang a session indefinitely.

In addition, claude.ts:1944–1966 measures `STALL_THRESHOLD_MS = 30_000` gaps
between events after the first chunk (to avoid conflating TTFB with stalls) and
emits `tengu_streaming_stall` telemetry. A summary is logged at end of stream
(claude.ts:2367–2380).

After the for-await loop exits, if `streamIdleAborted` is set the generator
throws `'Stream idle timeout - no chunks received'` (claude.ts:2334), which
drops into the catch block that triggers non-streaming fallback.

---

## 3. The event dispatcher — core accumulator

The main loop is `for await (const part of stream)` at `claude.ts:1940` with a
switch on `part.type`. Pre-loop state (reset at claude.ts:1859–1866):

| Name | Type | Purpose |
|---|---|---|
| `partialMessage` | `BetaMessage \| undefined` | Captured at `message_start`; used as a template for every `AssistantMessage` yielded at `content_block_stop`. |
| `contentBlocks` | `BetaContentBlock[]` | Per-index accumulator for text / tool_use / thinking / server_tool_use. |
| `newMessages` | `AssistantMessage[]` | Yielded messages, retained as references so `message_delta` can mutate their `usage`/`stop_reason` in place. |
| `usage` | `NonNullableUsage` | Running usage; updated by `updateUsage()` on each `message_start` / `message_delta`. |
| `stopReason` | `BetaStopReason \| null` | Final value arrives in `message_delta.delta.stop_reason`. |
| `ttftMs` | `number` | Time-to-first-token from `message_start`. |
| `research` | `unknown` | Internal-only field passed through on ant builds. |
| `isAdvisorInProgress` | `boolean` | Gate for ant-only advisor-tool telemetry. |

### 3.1 `message_start` (claude.ts:1980–1993)

```ts
partialMessage = part.message
ttftMs = Date.now() - start
usage = updateUsage(usage, part.message?.usage)   // output_tokens: 0 at this point
```

Input-token counts arrive here; output counts come later in `message_delta`.
`updateUsage` (claude.ts:2924) uses a "non-null, non-zero only" guard so the
output_tokens:0 in `message_start` doesn't clobber a later real value (comment:
claude.ts:2920–2923).

### 3.2 `content_block_start` (claude.ts:1995–2051)

Per-block-type initialization — **important because the SDK emits `input` as a
string during streaming but as an object during non-streaming**, and the code
compensates differently per type:

- **`tool_use`**: `{ ...part.content_block, input: '' }` — input is a string
  accumulator for `input_json_delta` deltas.
- **`server_tool_use`**: `{ ...part.content_block, input: '' as object }` —
  same string accumulator, cast at the type level. Advisor-tool (ant-only)
  fires `tengu_advisor_tool_call` telemetry (claude.ts:2008–2017).
- **`text`**: `{ ...part.content_block, text: '' }` — comment
  (claude.ts:2022–2026) notes the SDK sometimes ships initial text in
  `content_block_start` and then duplicates it in the delta, so the accumulator
  is zeroed.
- **`thinking`**: `{ ...part.content_block, thinking: '', signature: '' }` —
  signature is pre-initialized even though it only arrives via
  `signature_delta`, so downstream code can rely on the field existing.
- **default**: shallow spread `{ ...part.content_block }` — the comment at
  claude.ts:2040–2042 explains the SDK mutates its own blocks internally, so a
  copy is required for immutability. `advisor_tool_result` clears
  `isAdvisorInProgress`.

### 3.3 `content_block_delta` (claude.ts:2053–2169)

Multi-type accumulation. Every handler verifies the block's `type` matches the
delta's `type`, logs `tengu_streaming_error` + throws on mismatch:

| Delta | Handling | Guard |
|---|---|---|
| `text_delta` | `contentBlock.text += delta.text` | block.type === `text` |
| `input_json_delta` | `contentBlock.input += delta.partial_json` (string concat) | block.type ∈ {`tool_use`, `server_tool_use`} AND `typeof input === 'string'` |
| `thinking_delta` | `contentBlock.thinking += delta.thinking` | block.type === `thinking` |
| `signature_delta` | `contentBlock.signature = delta.signature` | block.type === `thinking` OR (gated `connector_text`) |
| `citations_delta` | **no-op** — `// TODO: handle citations` at claude.ts:2084–2085 | — |
| `connector_text_delta` | `contentBlock.connector_text += delta.connector_text` | feature-gated `CONNECTOR_TEXT` |

Key intricacy: **`input_json_delta` accumulates raw character-by-character text
without intermediate parsing.** Malformed partial JSON is never validated here;
the only parse happens in `normalizeContentFromAPI` after
`content_block_stop`. This is the deliberate alternative to `partialParse()`
(§2.1).

Research capture on ant builds: `research` is updated from every event type
that might carry it (claude.ts:2164–2168, 1986–1992, 2219–2227).

### 3.4 `content_block_stop` (claude.ts:2171–2211)

When a block finishes, a complete `AssistantMessage` is constructed and yielded
**for that block alone**:

```ts
// claude.ts:2192-2211
const m: AssistantMessage = {
  message: {
    ...partialMessage,
    content: normalizeContentFromAPI([contentBlock] as BetaContentBlock[], tools, options.agentId),
  },
  requestId: streamRequestId ?? undefined,
  type: 'assistant',
  uuid: randomUUID(),
  timestamp: new Date().toISOString(),
  ...(process.env.USER_TYPE === 'ant' && research !== undefined && { research }),
  ...(advisorModel && { advisorModel }),
}
newMessages.push(m)
yield m
```

This is the **first point at which a tool_use block becomes fully parsed and
actionable** — see §5. Note the `content` array holds **only the one just-
completed block**, not the full accumulated message. The consumer stitches
individual blocks back together as it iterates.

### 3.5 `message_delta` (claude.ts:2213–2293)

Final usage + stop_reason plus stop-reason side effects:

```ts
// claude.ts:2242-2256
stopReason = part.delta.stop_reason
const lastMsg = newMessages.at(-1)
if (lastMsg) {
  lastMsg.message.usage = usage         // direct mutation, NOT replacement
  lastMsg.message.stop_reason = stopReason
}
const costUSDForPart = calculateUSDCost(resolvedModel, usage)
costUSD += addToTotalSessionCost(costUSDForPart, usage, options.model)
```

The **direct property mutation** is deliberate and documented at
claude.ts:2236–2241: the transcript write queue holds a reference to
`message.message` and serializes it lazily on a 100ms flush interval. Object
replacement would decouple the queued reference from the final values. This is
a subtle but critical invariant; §14.

Stop-reason side effects all live here:

- **`refusal`** → `getErrorMessageIfRefusal()` (errors.ts:1184) yields a
  synthesized `SystemAPIErrorMessage` explaining the refusal and suggesting
  `/model claude-sonnet-4-20250514` if the refused model wasn't already it.
- **`max_tokens`** → logs `tengu_max_tokens_reached`, yields error with
  `CLAUDE_CODE_MAX_OUTPUT_TOKENS` hint.
- **`model_context_window_exceeded`** → logs
  `tengu_context_window_exceeded`, yields a recovery-hint error that
  **reuses the `max_output_tokens` recovery path** because from the model's
  perspective both mean "response cut off, continue" (claude.ts:2284–2291).

### 3.6 `message_stop` (claude.ts:2295)

No-op. `message_delta` already carried everything that matters.

### 3.7 Raw event fanout (claude.ts:2299–2303)

Every iteration of the loop — after the switch — yields a `stream_event`
passthrough:

```ts
yield {
  type: 'stream_event',
  event: part,
  ...(part.type === 'message_start' ? { ttftMs } : undefined),
}
```

Consumers: `cli/print.ts:904` (filters for `-p` JSON output), `HybridTransport`
(`cli/transports/HybridTransport.ts:118`), `ccrClient` for coalescing
(`cli/transports/ccrClient.ts:86–184`), `WebSearchTool` (tracks
`input_json_delta` for query extraction, `WebSearchTool.ts:305–365`), and the
SDK adapter for remote sessions
(`remote/sdkMessageAdapter.ts:45–218`).

---

## 4. Post-parse normalization — `normalizeContentFromAPI`

Location: `src/utils/messages.ts:2651–2751`.

Called **once per content block** at `content_block_stop` (claude.ts:2195) and
once at the non-streaming fallback (claude.ts:2574, 2672). Per-type:

### 4.1 `tool_use` (messages.ts:2661–2719)

1. Validate input is string or object (throws otherwise —
   messages.ts:2662–2668).
2. If string: `safeParseJSON(contentBlock.input)`; fall back to `{}` on parse
   failure and log `tengu_tool_input_json_parse_fail` with raw prefix in debug
   log (messages.ts:2676–2694). Known hole: recursive stringified JSON — the
   comment at messages.ts:2670–2674 notes nested stringified fields are still
   not handled.
3. Apply tool-specific normalization via `normalizeToolInput(tool,
   normalizedInput, agentId)` in a try/catch that preserves original input on
   failure (messages.ts:2700–2714). `normalizeToolInput` is per-tool; examples
   include Bash command trimming, Read path canonicalization, etc.

### 4.2 `text` (messages.ts:2721–2730)

Returned **as-is** to preserve exact bytes for prompt caching — the comment
warns that even whitespace edits break cache hits. Empty text gets a
`tengu_model_whitespace_response` telemetry event but is not dropped.

### 4.3 `server_tool_use` (messages.ts:2737–2745)

If input is string, `safeParseJSON ?? {}`; else passthrough.

### 4.4 Passthrough

`thinking`, `redacted_thinking`, `image`, `document`,
`code_execution_tool_result`, `mcp_tool_use`, `mcp_tool_result`,
`container_upload` — returned unchanged (messages.ts:2731–2748).

---

## 5. Tool-call extraction and streaming dispatch

The consumer is `query.ts:659–863`, which iterates over the generator and
extracts tool uses as each `AssistantMessage` arrives.

### 5.1 The consumer loop (query.ts:826–862)

```ts
// query.ts:826-845
if (message.type === 'assistant') {
  assistantMessages.push(message)

  const msgToolUseBlocks = message.message.content.filter(
    content => content.type === 'tool_use',
  ) as ToolUseBlock[]
  if (msgToolUseBlocks.length > 0) {
    toolUseBlocks.push(...msgToolUseBlocks)
    needsFollowUp = true   // the sole loop-exit signal
  }

  if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
    for (const toolBlock of msgToolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, message)
    }
  }
}
```

`needsFollowUp` is the flag that drives **whether the outer while loop calls
the API again** (query.ts:1062) — a tool_use in any yielded message sets it.
The comment at query.ts:554 notes `stop_reason === 'tool_use'` is **unreliable
and not used**; the block-presence check is authoritative.

### 5.2 `StreamingToolExecutor` (src/services/tools/StreamingToolExecutor.ts)

This class dispatches tools **as they stream in**, before the full assistant
message completes. Per-tool state:

```ts
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

Concurrency policy (StreamingToolExecutor.ts:126–151):

- Concurrent-safe tools may run **in parallel with other concurrent-safe
  tools**.
- Non-concurrent tools require **exclusive access** and **halt queue
  processing** until they run, to preserve submission order.
- `isConcurrencySafe(parsedInput)` is tool-specific and defaults to `false` if
  the input fails Zod validation.

Cascading abort (`siblingAbortController`, StreamingToolExecutor.ts:43–62,
347–363):

- Only Bash errors trigger sibling abort — other tools' errors don't cascade
  because Read/WebFetch/etc. are independent. Bash commands often have
  implicit dependency chains (`mkdir fails → later commands pointless`).
- Permission-dialog rejection bubbles up to the parent query controller via
  the abort listener at StreamingToolExecutor.ts:304–318 — without bubble-up,
  ExitPlanMode regresses (#21056).

Synthetic error reasons (StreamingToolExecutor.ts:153–205):

- `'user_interrupted'` → `REJECT_MESSAGE` with memory-correction hint; UI
  shows "User rejected edit" instead of "Error".
- `'streaming_fallback'` → tombstone-style error when executor is
  `.discard()`ed during streaming fallback.
- `'sibling_error'` → "Cancelled: parallel tool call X errored".

Result yielding (query.ts:847–862):

```ts
for (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message
    toolResults.push(
      ...normalizeMessagesForAPI([result.message], toolUseContext.options.tools)
        .filter(_ => _.type === 'user'),
    )
  }
}
```

`getCompletedResults()` (StreamingToolExecutor.ts:412+) is **non-blocking** —
it yields completed results and pending progress messages but doesn't wait for
in-flight tools. Progress messages are yielded immediately, independent of
ordering.

### 5.3 Unknown-tool handling (StreamingToolExecutor.ts:77–102)

Tools not found by `findToolByName` get a synthetic
`<tool_use_error>Error: No such tool available: ${block.name}</tool_use_error>`
result immediately, with `isConcurrencySafe: true` so it doesn't block the
queue. `status = 'completed'` — no execution attempted.

---

## 6. Stop reasons and control-flow decisions

From `BetaStopReason`:

| `stop_reason` | Where handled | Action |
|---|---|---|
| `end_turn` | query.ts:1062 (via `needsFollowUp=false`) | Exit streaming loop, run stop hooks, return. |
| `tool_use` | **Not read directly.** `needsFollowUp` is set when tool_use blocks appear. | Continue to next API iteration after tools run. |
| `max_tokens` | claude.ts:2266–2277 | Yield error message, hit `max_output_tokens` recovery path in query.ts (continues conversation using partial response). |
| `stop_sequence` | Not specifically handled — falls through as end_turn. | — |
| `pause_turn` | Not specifically handled — rare. | — |
| `refusal` | claude.ts:2258–2264 via `getErrorMessageIfRefusal` (errors.ts:1184) | Synthesize AUP message + model suggestion, yield, return. |
| `model_context_window_exceeded` | claude.ts:2279–2292 | Yield error, reuse `max_output_tokens` recovery. |

Observation: **the CLI's loop-exit decision depends on content-block presence,
not `stop_reason`.** The flag at `query.ts:554–557` explicitly calls out that
`stop_reason === 'tool_use'` is not always set correctly.

---

## 7. Thinking / extended-thinking handling

### 7.1 Request-side configuration (claude.ts:1596–1629)

- Disabled entirely if `CLAUDE_CODE_DISABLE_THINKING` is truthy.
- For adaptive-thinking models: `{ type: 'adaptive' }` with no budget.
- Else: `{ type: 'enabled', budget_tokens: N }` where `N` is capped at
  `max_tokens - 1`.

### 7.2 Response-side accumulation

- `content_block_start` of `thinking` initializes `thinking: ''` and
  `signature: ''` (claude.ts:2030–2037).
- `thinking_delta` appends to `.thinking` (claude.ts:2148–2160).
- `signature_delta` sets `.signature` (claude.ts:2127–2146). The order is not
  guaranteed, hence the defensive init.

### 7.3 Preservation invariants

- `normalizeContentFromAPI` **returns thinking blocks unchanged** (default
  branch, messages.ts:2747) so `signature` survives the normalize round-trip.
- On model fallback, thinking signatures are **model-bound**: replaying a
  protected-thinking block (e.g. capybara) to an unprotected fallback (e.g.
  opus) 400s the API. `stripSignatureBlocks(messagesForQuery)` is called
  before retry on ant builds (query.ts:927–929).
- On streaming fallback, partial thinking blocks are **tombstoned**
  (query.ts:713–723) because their signatures are invalid — replaying them
  would error with "thinking blocks cannot be modified".

---

## 8. Streaming UI integration

Three levels of progressive surface:

1. **Raw SSE events** — every consumer that subscribes to `stream_event`
   messages sees `BetaRawMessageStreamEvent` directly. `cli/print.ts:904`
   filters them out of `-p` JSON output unless explicitly requested; the TUI
   lights up tool-progress indicators on `message_start`.
2. **Per-block assistant messages** — yielded at `content_block_stop`, each
   carries one completed block. The TUI renders them incrementally: text
   block → print once, tool_use block → start showing tool progress, thinking
   block → show thinking indicator (gated on settings).
3. **Tool-progress messages** — `StreamingToolExecutor.pendingProgress`
   surfaces intermediate tool events (e.g. Bash output streaming lines) via
   `getCompletedResults()` without waiting for the tool to finish.

Thinking-block gating happens at the display layer, not in the response
pipeline — blocks are always in the message content.

---

## 9. Usage tracking and cost calculation

### 9.1 Incremental accumulation

- `updateUsage()` (claude.ts:2924–2987) — merges a `BetaMessageDeltaUsage`
  into the running `NonNullableUsage`. The "non-null and > 0" guard on
  `input_tokens`, `cache_*`, `server_tool_use.*`, and `cache_deleted` fields
  prevents a `message_delta` with explicit 0 from clobbering the real value
  from `message_start`. Uses `partUsage.output_tokens ?? usage.output_tokens`
  (no > 0 check) because the final delta legitimately reports the final
  output_tokens.
- `accumulateUsage()` (claude.ts:2993–3038) — sums across multiple messages
  in a session. Summed: `input_tokens`, `cache_creation_input_tokens`,
  `cache_read_input_tokens`, `output_tokens`, `server_tool_use.*`,
  `cache_creation.ephemeral_*_input_tokens`, `cache_deleted_input_tokens`
  (feature-gated `CACHED_MICROCOMPACT`). Latest-wins: `service_tier`,
  `inference_geo`, `iterations`, `speed`.

### 9.2 Hooks and side channels

- `cost-tracker.ts:271` reads `usage.server_tool_use?.web_search_requests`
  for per-search billing.
- `checkResponseForCacheBreak(...)` (claude.ts:2382–2392, gated
  `PROMPT_CACHE_BREAK_DETECTION`) fires off cache-break detection based on
  `cache_read_input_tokens` / `cache_creation_input_tokens`.
- `extractQuotaStatusFromHeaders(resp.headers)` (claude.ts:2400) pulls quota
  warnings from response headers after the stream finishes.

### 9.3 Field not in SDK type

`cache_deleted_input_tokens` is real but **not in the SDK's
`BetaMessageDeltaUsage`**. It's read via `as unknown as { ... }` casts and
kept off `NonNullableUsage` so the string can be dead-code-eliminated from
external builds (comment: claude.ts:2965–2969, 2022–2023).

---

## 10. Error handling on malformed / interrupted responses

### 10.1 Partial JSON on abort

If the stream ends mid-`input_json_delta`, `content_block_stop` never arrives
for that tool_use block. The stream-completion check at claude.ts:2350–2364
catches this:

```ts
if (!partialMessage || (newMessages.length === 0 && !stopReason)) {
  logEvent('tengu_stream_no_events', { ... })
  throw new Error('Stream ended without receiving any events')
}
```

Two failure modes are conflated here, both treated as "fall back to
non-streaming":

1. No events at all (`!partialMessage`) — proxy returned 200 with non-SSE body.
2. Partial events — `message_start` arrived but `content_block_stop` and
   `message_delta` never did.

The `stopReason` check avoids a false positive for structured output
(`--json-schema`): turn-1 calls `StructuredOutput` tool, turn-2 legitimately
responds with `end_turn` and zero content blocks. Without the `stopReason`
guard this would trigger fallback on every structured-output turn.

### 10.2 Non-streaming fallback

Triggered from the catch block at claude.ts:2404–2569. Env/feature escape
hatch `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` or
`tengu_disable_streaming_to_non_streaming_fallback` disables it
(claude.ts:2469–2502). The comment at claude.ts:2464–2468 explains **why**
disabling exists: during streaming-tool-execution, the partial stream may have
started a tool, and the non-streaming retry produces the same tool_use again,
double-executing it (inc-4258).

The fallback:

1. Calls `executeNonStreamingRequest()` (claude.ts:818–905) — wraps the
   SDK's non-streaming `.create()` in `withRetry`, with per-attempt timeout
   of 120s (remote) or 300s (local).
2. Sets `didFallBackToNonStreaming = true` and calls
   `options.onStreamingFallback()` — which `query.ts:678–680` uses to set
   `streamingFallbackOccured = true`.
3. Receives a full `BetaMessage`, runs it through `normalizeContentFromAPI`,
   yields one synthesized `AssistantMessage`.

On the query.ts side, when `streamingFallbackOccured` is true
(query.ts:712–740):

- All already-yielded assistant messages are **tombstoned** (their partial
  thinking signatures would 400 on replay).
- Tool results and tool-use blocks are cleared.
- `needsFollowUp` is reset.
- The `streamingToolExecutor` is **`.discard()`d and replaced** to prevent
  old tool_use_ids from leaking into retries.

### 10.3 404 on stream creation

Handled separately at claude.ts:2612–2646 — gateway 404s during
`.withResponse()` are thrown before the stream opens, so the normal catch
block doesn't see them. This path triggers the same non-streaming fallback
without the tombstoning dance (no partial stream to clean up).

### 10.4 User abort vs SDK timeout

Both throw `APIUserAbortError` but differ on `signal.aborted`
(claude.ts:2434–2462):

- `signal.aborted === true` → real user ESC; rethrow (query.ts handles it
  via the abort-branch at query.ts:1030–1051).
- `signal.aborted === false` → SDK internal timeout; rethrow as
  `APIConnectionTimeoutError` so `withRetry` treats it as a retryable
  connection error.

### 10.5 Model fallback (`FallbackTriggeredError`)

Thrown from inside the retry generator when the API signals "model under
load". Caught at query.ts:893–950:

- Model switches to `fallbackModel`.
- Assistant messages and tool state reset.
- Executor discarded and recreated.
- Thinking signatures stripped on ant builds.
- User sees a warning-level system message.
- Streaming retries on the new model.

---

## 11. Provider abstraction — or lack thereof

There is **no multi-provider adapter layer inside `queryModel`**. The code
speaks `@anthropic-ai/sdk` types directly (`BetaRawMessageStreamEvent`,
`BetaMessage`, `BetaStopReason`, `BetaContentBlock`, etc.). Provider-specific
behavior (Bedrock, Vertex, CCR, grove) is handled upstream:

- `getAnthropicClient({ source })` returns the right client + base URL +
  header injection for the provider (not shown here).
- `normalizeModelStringForAPI(...)` (claude.ts:867) strips or rewrites
  model names where providers differ.
- First-party-only features (like `CLIENT_REQUEST_ID_HEADER` injection at
  claude.ts:1813–1816) are guarded on `getAPIProvider() === 'firstParty'`.

The **VCR layer** (`src/services/vcr.ts`, `withStreamingVCR` at
vcr.ts:88–160) is the only generic wrapper — it records live streams to
fixture files and replays them during test runs. It dehydrates file-contents
fields for reproducibility and adds cached cost back to the session tracker.

The **remote-session adapter** (`src/remote/sdkMessageAdapter.ts`) translates
SDK-level `SDKPartialAssistantMessage` objects from the daemon into the same
`{ type: 'stream_event', event }` shape the streaming generator produces, so
downstream consumers can't tell the difference between local streaming and
remote replay. `convertStreamEvent()` (sdkMessageAdapter.ts:45) is the
bidirectional bridge.

---

## 12. How responses become the next turn

Back in `query.ts`, after the streaming loop ends (query.ts:864):

1. **`needsFollowUp` check** (query.ts:1062): if false, exit streaming loop
   and run stop hooks; the outer while returns.
2. **Tool-result assembly**: by the time the loop ends, `toolResults` is
   already populated (from `StreamingToolExecutor.getCompletedResults()`
   yields during streaming). Tools that hadn't completed during streaming get
   awaited here via additional `getRemainingResults()` (not shown, but
   referenced in StreamingToolExecutor.ts:50).
3. **Next-turn message construction**: tool_result blocks are wrapped into a
   synthetic user message and appended to `messagesForQuery`, which feeds
   the next `queryModel()` call. Because `messagesForQuery` is prepended with
   byte-stable `userContext` (query.ts:660, `prependUserContext`), prompt
   caching works across turns.
4. **Recovery paths** on withheld errors:
   - Context-collapse drain (query.ts:1089–1117) — `contextCollapse.recoverFromOverflow()`.
   - Reactive compact (query.ts:1119–1175) — `reactiveCompact.tryReactiveCompact()`.
   - `maxOutputTokens` recovery — injects continuation prompt and retries with
     a higher `max_tokens` budget.
5. **Microcompact boundary** (query.ts:866–892, gated `CACHED_MICROCOMPACT`)
   — reads `usage.cache_deleted_input_tokens` and yields a synthetic
   `microcompact_boundary` message so the transcript records exactly how many
   tokens the cache editing deleted.

---

## 13. Data-structure & code-reference quick index

| Concern | File:line |
|---|---|
| `queryModel()` generator | `src/services/api/claude.ts:1017` |
| `queryModelWithStreaming()` VCR wrapper | `src/services/api/claude.ts:752–780` |
| Raw stream create + TTFB checkpoint | `src/services/api/claude.ts:1818–1836` |
| Idle watchdog setup | `src/services/api/claude.ts:1868–1929` |
| For-await main loop | `src/services/api/claude.ts:1940–2304` |
| `message_start` | `src/services/api/claude.ts:1980–1993` |
| `content_block_start` (per-type init) | `src/services/api/claude.ts:1995–2051` |
| `content_block_delta` (delta switch) | `src/services/api/claude.ts:2053–2169` |
| `content_block_stop` (message assembly) | `src/services/api/claude.ts:2171–2211` |
| `message_delta` (stop_reason side effects) | `src/services/api/claude.ts:2213–2293` |
| Raw event passthrough yield | `src/services/api/claude.ts:2299–2303` |
| Incomplete-stream detection | `src/services/api/claude.ts:2350–2364` |
| Non-streaming fallback trigger | `src/services/api/claude.ts:2404–2569` |
| 404 stream-creation fallback | `src/services/api/claude.ts:2612–2646` |
| `updateUsage` | `src/services/api/claude.ts:2924–2987` |
| `accumulateUsage` | `src/services/api/claude.ts:2993–3038` |
| `normalizeContentFromAPI` | `src/utils/messages.ts:2651–2751` |
| `getErrorMessageIfRefusal` | `src/services/api/errors.ts:1184–1207` |
| `executeNonStreamingRequest` | `src/services/api/claude.ts:818–905` |
| Consumer loop, `needsFollowUp`, executor dispatch | `src/query.ts:551–863` |
| Streaming fallback tombstoning | `src/query.ts:712–740` |
| Model fallback recovery | `src/query.ts:893–950` |
| `StreamingToolExecutor` class | `src/services/tools/StreamingToolExecutor.ts` |
| `addTool` / `processQueue` / concurrency gate | `src/services/tools/StreamingToolExecutor.ts:76–151` |
| Synthetic error messages | `src/services/tools/StreamingToolExecutor.ts:153–205` |
| `executeTool` (child abort, tool generator) | `src/services/tools/StreamingToolExecutor.ts:265–405` |
| `getCompletedResults` | `src/services/tools/StreamingToolExecutor.ts:412+` |
| VCR adapter | `src/services/vcr.ts:88–170` |
| Remote SDK adapter | `src/remote/sdkMessageAdapter.ts:45–218` |
| CCR coalesced stream | `src/cli/transports/ccrClient.ts:86–184, 736` |

---

## 14. Intricate patterns flagged for follow-up

The deliverable asks to flag if streaming or parsing logic is intricate. It
**is**, and the below subsystems warrant dedicated deep dives:

1. **`normalizeContentFromAPI` + `normalizeToolInput` coupling** —
   per-tool normalization rules (Bash trimming, Read canonicalization, etc.)
   live in many tool files. The recursive-stringified-JSON hole
   (messages.ts:2670–2674) is open. Worth a focused audit of what each tool's
   normalizer does and whether the `{}` fallback masks real bugs.

2. **Streaming tool execution + fallback interaction** — the dance between
   `StreamingToolExecutor.discard()`, fresh executor recreation, and the
   `streamingFallbackOccured` / `onStreamingFallback` signal path across
   `claude.ts` and `query.ts` is subtle. Inc-4258 (double-execution) and the
   `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` escape hatch both point to
   ongoing correctness work. Tombstoning invariants for thinking signatures
   are model-specific.

3. **Idle watchdog + stall detection + abort propagation** — three
   interacting timers with instrumentation probes (`exit_delay_ms`,
   `cli_stream_loop_exited_after_watchdog_*`). Exit-path correctness
   (`clean` vs `error`) gates telemetry; subtle bugs here are hard to
   reproduce.

4. **Stop-reason → recovery path mapping** — `max_tokens`,
   `model_context_window_exceeded`, prompt-too-long 413, media-size errors,
   `refusal` all have different recovery paths (context-collapse drain,
   reactive compact, model fallback, max-output-token recovery) that interact
   through `query.ts`'s `state` machine. The comments at
   query.ts:622–647 and 1065–1175 are dense; the full state-machine would
   benefit from a diagram.

5. **Direct-mutation-for-transcript invariant** — the pattern at
   claude.ts:2236–2248 (mutate instead of replace so the 100ms flush queue
   sees final values via reference) is load-bearing and easy to break in
   refactors. Any contributor who tries to "clean up" that code with a
   spread-replace breaks transcript finalization.

6. **Usage merge policy** — the "non-null and > 0" guards in `updateUsage`
   vs. the latest-wins semantics in `accumulateUsage` vs. the SDK's
   undocumented `cache_deleted_input_tokens` field (kept off the type for
   DCE reasons) — a dedicated reference on the usage-merge decision tree
   would save confusion.

7. **VCR / remote adapter / CCR transport** — three different places where
   `BetaRawMessageStreamEvent` is normalized in or out. Their invariants need
   to stay in sync or replay diverges from live.

---

## 15. Summary — "how API responses become actions"

The API response pipeline is a **per-block streaming accumulator** with an
**intentional raw-stream bypass** of the SDK's partial-JSON parser, assembling
complete content blocks into `AssistantMessage`s that are yielded **as each
block closes** (not at message end), with tool calls dispatched immediately
into a **concurrent-safe execution queue** that runs tools in parallel with
further streaming. The outer `query.ts` loop converts tool_result blocks back
into the next user message, with `needsFollowUp` (content-block presence, not
`stop_reason`) as the only authoritative "continue?" signal. Stop-reason side
effects (refusal, max_tokens, context-window) synthesize error messages;
streaming failures fall back to non-streaming via tombstoning + retry;
signatures on thinking blocks are model-bound and stripped before model
fallback to avoid 400s.

**This is intricate enough to warrant the flag** — the comments in `claude.ts`
alone reference at least five post-incident fixes (inc-3694, inc-4029,
inc-4258, gh-#21056, gh-#1513) whose invariants the current code is
structured around.
