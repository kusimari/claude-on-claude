# 09 — Claude API Communication Patterns

**Scope**: Technical details of how Claude Code talks to the Anthropic
API — request construction, streaming, retry, error taxonomy, prompt
caching, and multi-provider abstraction. Implementation lives in
`src/services/api/*.ts` (~11.2k LOC across ~20 files) plus
`src/utils/api.ts` (718 LOC) and `src/utils/tokens.ts`.

The three biggest files are `claude.ts` (3,419 LOC — the main request
pipeline), `errors.ts` (1,207 LOC — error taxonomy), and `withRetry.ts`
(822 LOC — retry policy). The layer is fed by prompt construction
upstream (report 08) and consumed by `query.ts` / `QueryEngine.ts`
downstream (report 07); this report covers only the API boundary.

---

## 1. Client construction (`services/api/client.ts`, 389 LOC)

`getAnthropicClient()` is the single entry point. It is called **fresh
per request** (not memoized) so that OAuth tokens, AWS creds, and
Vertex creds all refresh naturally every call.

Four providers, auto-selected by env var:

| Provider | Flag | SDK |
|---|---|---|
| First-party Anthropic | (default) | `Anthropic` |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK=true` | `AnthropicBedrock` |
| Google Vertex | `CLAUDE_CODE_USE_VERTEX=true` | `AnthropicVertex` |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY=true` | `AnthropicFoundry` |

Common `ARGS` bag (`client.ts:141-152`): `defaultHeaders`, `maxRetries`,
`timeout` (default 600s, env `API_TIMEOUT_MS`),
`dangerouslyAllowBrowser: true`, `fetchOptions` (proxy + mTLS via
`getProxyFetchOptions()`), and optional `fetch` override.

### 1.1 Provider-specific quirks

- **Bedrock** (`client.ts:153-189`): per-model region (e.g.
  `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` for Haiku), bearer token via
  `AWS_BEARER_TOKEN_BEDROCK`, skip-auth via
  `CLAUDE_CODE_SKIP_BEDROCK_AUTH`, credential refresh through
  `refreshAndGetAwsCredentials()`.
- **Vertex** (`client.ts:221-298`): per-model region env vars (e.g.
  `VERTEX_REGION_CLAUDE_3_5_SONNET`), `GoogleAuth` with
  `scopes: ['cloud-platform']`. Pre-seeds `projectId` from
  `ANTHROPIC_VERTEX_PROJECT_ID` to avoid a 12s metadata-server timeout
  (`client.ts:273-287`).
- **Foundry** (`client.ts:191-219`): `ANTHROPIC_FOUNDRY_RESOURCE` or
  `ANTHROPIC_FOUNDRY_BASE_URL`; Azure AD via `DefaultAzureCredential`
  + `getBearerTokenProvider()`.
- **First-party** (`client.ts:300-315`): picks `authToken` (OAuth) if
  `isClaudeAISubscriber()` else `apiKey` (env or helper). Staging OAuth
  uses an alternate `baseURL`.

### 1.2 Headers

Standard: `x-app: cli`, `User-Agent`,
`X-Claude-Code-Session-Id`, `x-claude-remote-container-id`,
`x-claude-remote-session-id`, `x-client-app` (`client.ts:105-116`).
Extras: `ANTHROPIC_CUSTOM_HEADERS` env is parsed as multi-line
`Name: Value` and injected (`client.ts:330-354`).
`x-anthropic-additional-protection` flag-gated (`client.ts:124-129`).

### 1.3 Per-request correlation IDs

`client.ts:358-389` wraps fetch to inject a UUID
`x-client-request-id` **only for first-party API**. The rationale
(explained in comments): server-generated request IDs never come back
on timeouts, so a client-generated ID correlates to server logs.
Callers can override by pre-setting the header. Debug-logged with
source.

### 1.4 The "lying cast" pattern (`client.ts:188-189, 296-297`)

Bedrock/Vertex/Foundry SDKs are cast to `Anthropic` at compile time
even though they don't expose `batches` or `models` APIs. Comments say
so explicitly: "we have always been lying about the return type." Safe
at runtime because only `beta.messages.stream()` / `messages.create()`
are ever called downstream; the cast keeps the rest of the codebase
provider-agnostic.

---

## 2. Request pipeline (`services/api/claude.ts`, 3,419 LOC)

### 2.1 Entry points

| Function | Location | Purpose |
|---|---|---|
| `queryModel()` | `claude.ts:1017` | Main async-generator pipeline; all other entry points go through it |
| `queryModelWithStreaming()` | `claude.ts:752` | Public async-generator wrapper, yields `StreamEvent \| AssistantMessage \| SystemAPIErrorMessage` |
| `queryModelWithoutStreaming()` | `claude.ts:709` | Drains the generator, returns final `AssistantMessage`; used by non-agent callers |
| `queryWithModel()` | `claude.ts:3300` | Public helper for out-of-loop queries (commands, agent generation) |
| `queryHaiku()` | `claude.ts:3241` | Small-model fast-path (titles, summaries, parsing); forces `getSmallFastModel()`, no thinking/tools/streaming |
| `verifyApiKey()` | `claude.ts:530` | `/login` dry-run; one-token request to validate creds |
| `stripExcessMediaItems()` | `claude.ts:956` | Dedupes to 4-image API limit for many-image requests |

Support helpers: `updateUsage()` (`claude.ts:2924`) merges streaming
usage deltas; `accumulateUsage()` (`claude.ts:2993`) sums across turns;
`cleanupStream()` (`claude.ts:2898`) aborts the stream controller on
error/timeout; `addCacheBreakpoints()` / `buildSystemPromptBlocks()`
(around `claude.ts:3063`) inject `cache_control: {ttl, type}` markers.

### 2.2 Parameter construction

- **System prompt blocks** (`claude.ts:1207-1264`) — per-block
  cache-control scope (global/org) and TTL (5m / 1h).
- **Betas** (`claude.ts:1071`) — merged from model-specific, provider-
  specific, and feature-specific sources (advisor, cache editing,
  structured outputs, tool search, fast mode, thinking, effort, task
  budgets).
- **Tools** (`claude.ts:1064-1172`) — schema via `toolToAPISchema()`
  (`utils/api.ts:119-160+`), deferred tool loading via tool search
  beta, MCP filtering.
- **Model resolution** (`claude.ts:1057-1062`) — resolves inference
  profile backing model for Bedrock.
- **Thinking / effort / fast-mode** (`claude.ts:1599-1645`) — model
  support gated via `modelSupportsThinking()`,
  `modelSupportsAdaptiveThinking()`; max budget from
  `getMaxThinkingTokensForModel()`.
- **Cache TTL** (`claude.ts:358-391`) — 5m vs 1h driven by source,
  overage status, thinking on/off.
- **Extra body** (`claude.ts:272`) — `CLAUDE_CODE_EXTRA_BODY` and
  `anthropic_internal` injection.

### 2.3 Streaming consumption

`queryModel()` reads the **raw** `BetaRawMessageStreamEvent` stream
(not the high-level wrapper) because the wrapper re-parses partial
JSON every chunk — O(n²) for large tool arguments
(`claude.ts:1818-1819`). Three event types drive state:

- `message_start` — initialize usage totals.
- `content_block_delta` — yield `StreamEvent` chunks; accumulate text
  and tool-use deltas.
- `message_delta` — final usage + stop reason.

### 2.4 Stop reasons (`claude.ts:2650-2750`)

`end_turn`, `tool_use`, `max_tokens`,
`model_context_window_exceeded` (special retry path — see §3),
`refusal` (yields an error message via `getErrorMessageIfRefusal()`
in `errors.ts:1184-1207`).

### 2.5 Usage extraction (`claude.ts:2680-2750`)

Cumulative `input_tokens`, `cache_creation_input_tokens`,
`cache_read_input_tokens`, `output_tokens`, plus server-side tool
counters (`web_search_requests`, `web_fetch_requests`), plus cache
breakdowns (`ephemeral_1h_input_tokens`,
`ephemeral_5m_input_tokens`, `cache_deleted_input_tokens`).

---

## 3. Retry logic (`services/api/withRetry.ts`, 822 LOC) ⚠️

Complex enough to warrant its own deep-dive — this section is a
summary, not a complete treatment.

### 3.1 Retryable taxonomy

- **408 / 409 / 500-504** — always retry.
- **429 rate limit** — retry unless ClaudeAI subscriber (enterprise
  only).
- **529 overload** — retry if foreground source; fail immediately for
  background sources (speculation, session memory, prompt suggestion)
  to avoid cascading amplification.
- **401 / 403 auth** — refresh token / creds, then retry. CCR mode
  (`CLAUDE_CODE_REMOTE=true`) treats 401/403 as **transient network
  hiccups** (JWT is infrastructure-provided, always valid) and
  bypasses the `x-should-retry:false` header directive
  (`withRetry.ts:712-717`).
- **max-tokens overflow** — special path, see §3.4.
- **`x-should-retry` server header** — honored, except ants on 5xx
  (`withRetry.ts:732-751`).

### 3.2 Backoff

Exponential `500ms * 2^(n-1)`, capped at 32s by default + 0-25% jitter
(`withRetry.ts:530-548`). `Retry-After` (seconds) and
`anthropic-ratelimit-unified-reset` (unix timestamp) override the
calculated delay.

### 3.3 Persistent mode (`withRetry.ts:96-104, 368-507`)

Ant-only `CLAUDE_CODE_UNATTENDED_RETRY`. Max per-attempt delay 5 min,
total cap 6 hours. Long sleeps chunked into 30s heartbeat yields
(system-message) so host infrastructure sees activity and doesn't
mark the session idle. Indefinite retry loop with a separate
`persistentAttempt` counter clamped at `maxRetries` for termination.

### 3.4 Max-tokens context overflow (`withRetry.ts:388-427`)

When API returns `400 "input length and max_tokens exceed ... X + Y >
Z"`, parse numbers, calculate `available = Z - X - 1000` (safety
buffer), floor at 3000 output tokens, retry with adjusted max_tokens.
If thinking is enabled, ensures `max_tokens > thinking.budget_tokens`
(API invariant) by capping the thinking budget first — graceful
degrade instead of hard fail.

### 3.5 Fast-mode fallback decision (`withRetry.ts:267-314`)

Dual path when fast mode hits 429/529:

- `Retry-After < 20s` — **keep fast mode on**, retry quickly. Same
  model name = same cache partition, preserves prompt cache.
- `Retry-After ≥ 20s` (or unknown) — **cooldown**: switch to
  standard-speed model, cache is lost but user wait time is bounded.
  Min cooldown 10 min to prevent flip-flop.

Logs `tengu_api_retry` per attempt (`withRetry.ts:468-475`).

### 3.6 Opus → Sonnet fallback (`withRetry.ts:326-365`)

After 3 consecutive 529s on foreground queries, gated by
`FALLBACK_FOR_ALL_PRIMARY_MODELS` or non-subscriber Opus users. Emits
`tengu_api_opus_fallback_triggered`, throws `FallbackTriggeredError`
which the UI catches and uses to prompt model switch.

### 3.7 Auth cache invalidation

On 401/403: `clearApiKeyHelperCache()` (`withRetry.ts:774`),
`clearAwsCredentialsCache()` / `clearGcpCredentialsCache()`
(`withRetry.ts:650-694`), force fresh client via `getClient()` again
(`withRetry.ts:250`). OAuth 403 `"token revoked"` calls
`handleOAuth401Error()` to refresh.

### 3.8 Transport-level retries

- **ECONNRESET / EPIPE** — detect via
  `extractConnectionErrorDetails()` (`withRetry.ts:112-118`), disable
  keep-alive pooling (`withRetry.ts:226-230`), reconnect next attempt.
- **Abort signal** checked at loop entry and in persistent-sleep
  chunks (`withRetry.ts:190-191, 491, 501`); throws `APIUserAbortError`
  immediately. Stream cleanup via `cleanupStream()`.

### 3.9 Streaming idle timeout (`claude.ts:1868-1926`)

2-minute watchdog: if no chunks arrive, abort and fall back to
non-streaming (`claude.ts:1768-1857`), provided max_tokens ≤ 64k.
Runs **during** streaming, not between retries, so catches stalled
connections.

---

## 4. Error taxonomy (`services/api/errors.ts`, 1,207 LOC)

`getAssistantMessageFromError()` (`errors.ts:425-934`) is the central
converter. It reads the thrown error, classifies it, and returns a
synthetic `AssistantMessage` with `isApiErrorMessage: true`, a
semantic `error` type, and optional `errorDetails` used by reactive
compact.

### 4.1 Classification codes (`errors.ts:965-1161`)

Auth: `invalid_api_key`, `token_revoked`, `oauth_org_not_allowed`,
`auth_error`, `ssl_cert_error`.
Capacity: `rate_limit`, `server_overload`, `repeated_529`,
`capacity_off_switch`.
Content: `prompt_too_long`, `image_too_large`, `pdf_too_large`,
`pdf_password_protected`, `pdf_invalid`.
Tools: `tool_use_mismatch`, `unexpected_tool_result`,
`duplicate_tool_use_id`.
Other: `invalid_model`, `invalid_request`, `client_error`,
`connection_error`, `api_timeout`, `server_error`, `unknown`.

### 4.2 User-facing strings (excerpt)

- `INVALID_API_KEY_ERROR_MESSAGE = "Not logged in · Please run /login"`.
- `TOKEN_REVOKED_ERROR_MESSAGE = "OAuth token revoked · Please run /login"`.
- `OAUTH_ORG_NOT_ALLOWED_ERROR_MESSAGE = "Your account does not have access to Claude Code. Please run /login."`.
- `CCR_AUTH_ERROR_MESSAGE = "Authentication error · This may be a temporary network issue, please try again"` (CCR 401/403 reframed as transient) (`errors.ts:817-822`).
- `PROMPT_TOO_LONG_ERROR_MESSAGE = "Prompt is too long"` + parsed token counts via `parsePromptTooLongTokenCounts()` (used by reactive compact).

### 4.3 Rate-limit unified headers (`errors.ts:465-558`)

New headers: `anthropic-ratelimit-unified-representative-claim`
(5hr / 7day / 7day_opus windows) and
`anthropic-ratelimit-unified-overage-status` (`allowed`,
`allowed_warning`, `rejected`). Forwarded to
`getRateLimitErrorMessage()` in `claudeAiLimits.ts` for
subscription-tier-specific wording.

### 4.4 Media-size detection (`errors.ts:133-153`)

Regex on message:
- image: `"image exceeds" && "maximum"`.
- many-image dimensions: `"image dimensions exceed" && "many-image"`.
- PDF: `/maximum of \d+ PDF pages/`.

`isMediaSizeErrorMessage()` lets reactive compact know to call
`stripImagesFromMessages()` and retry.

### 4.5 Connection error extraction (`errorUtils.ts:42-83`)

Walks `.cause` chain up to 5 levels deep (Anthropic SDK wraps root
errors). Known SSL codes (`errorUtils.ts:5-29`):
`UNABLE_TO_VERIFY_LEAF_SIGNATURE`, `CERT_HAS_EXPIRED`,
`DEPTH_ZERO_SELF_SIGNED_CERT`, `ERR_TLS_CERT_ALTNAME_INVALID`,
`ERR_TLS_HANDSHAKE_TIMEOUT`, ~12 more. Actionable hint (generic
template at `errorUtils.ts:94-100`): "set `NODE_EXTRA_CA_CERTS` to
your CA bundle, or ask IT to allowlist `*.anthropic.com`."

### 4.6 Message sanitization (`errorUtils.ts:107-260`)

CloudFlare HTML error pages stripped via DOCTYPE/tag detection
(`errorUtils.ts:122-130`); `<title>` tag preserved if present. After
JSONL deserialization (for `--resume`), the `.message` property may
be lost — nested extraction tries
`error.error.error.message` (standard) then `error.error.message`
(Bedrock) (`errorUtils.ts:169-198`).

---

## 5. Prompt-cache break detection (`services/api/promptCacheBreakDetection.ts`, 727 LOC)

Tracks per-source baseline state (hashed system prompt, per-tool
schema hashes, model, betas, effort, cache strategy, overage state)
and diffs against current on every query
(`promptCacheBreakDetection.ts:101-163`).

### 5.1 Break causes (`promptCacheBreakDetection.ts:28-99`)

System prompt text / cache scope / TTL changes, per-tool schema
hashes, model changes, fast-mode toggle, global-cache-strategy flip
(`tool_based` ↔ `system_prompt` ↔ `none`) when MCP connects/
disconnects, beta header set changes, effort value, extra-body hash.

Sticky-on invariants that should **not** break cache:
auto-mode-latched, overage-eligibility-latched, cache-editing-beta-
latched. These are verified against latched state so we can catch
regressions that silently unset them.

### 5.2 When it runs

Triggered per query after system prompt and tools are finalized,
keyed by `querySource` (or `agentId` for subagents). Cap of 10 tracked
sources. Haiku excluded (different cache semantics). Short-lived
sources (speculation, session_memory, prompt_suggestion) not tracked.

### 5.3 Reporting

Threshold: ≥ 2000-token drop triggers warning
(`promptCacheBreakDetection.ts:117-120`). TTL-driven drops (5m / 1h
expiry) attributed to server, not logged as breaks. Event:
`tengu_cache_break` with full pending-changes diff.

---

## 6. Logging (`services/api/logging.ts`, 788 LOC)

Lifecycle events:

- `tengu_api_request_started` — model, query source, tool count.
- `tengu_api_success` — tokens, duration, provider.
- `tengu_api_failure` — classification, status.
- `tengu_api_retry` — attempt, delay, status.
- `tengu_api_persistent_retry_wait` — for unattended mode.
- `tengu_api_opus_fallback_triggered`.
- `tengu_max_tokens_context_overflow_adjustment`.
- `tengu_cache_break`.

Sinks: Statsig, Datadog, local JSONL transcript (used by `--resume`),
optional OTEL spans (`isBetaTracingEnabled()`).

### 6.1 Redaction

Metadata is typed with a carry-the-bag assertion
`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`
(`logging.ts:34-36`). Never logged: user messages, assistant
responses, file paths, code, Authorization headers.

### 6.2 Error-message extraction for logging (`logging.ts:48-54`)

Walks the same nested structure as `errorUtils.ts` — top-level
`.message`, then `error.error.error.message` (standard), then
`error.error.message` (Bedrock), else type string.

---

## 7. Shared utilities

### `src/utils/api.ts` (718 LOC)

- `toolToAPISchema()` (`utils/api.ts:119-160+`) — Tool → `BetaTool`
  conversion with caching, `defer_loading` for tool search,
  `cache_control` injection.
- `normalizeFileEditInput()`, `stripTrailingWhitespace()` — FileEdit
  input normalization at the API boundary.
- `toolMatchesName()` — string vs name-object comparison.
- Types: `CacheScope` (`'global' | 'org'`), `SystemPromptBlock`.

### `src/utils/tokens.ts`

- `getTokenUsage()` / `getTokenCountFromUsage()` — extract/sum the
  four token categories.
- `tokenCountFromLastAPIResponse()` / `finalContextTokensFromLastResponse()`
  — context-window size from last usage, accounting for server-side
  tool iterations.

---

## 8. Integration surfaces (call-site inventory)

| Caller | Function | Use |
|---|---|---|
| `query.ts` / `QueryEngine.ts` | `queryModelWithStreaming()` | Main agent loop |
| `services/compact/compact.ts`, `microCompact.ts` | `queryModelWithStreaming()` | Conversation summarization |
| `services/awaySummary.ts` | `queryModelWithoutStreaming()` | Away-mode summary |
| `commands/rename/generateSessionName.ts` | `queryHaiku()` | Session title/rename |
| `utils/sessionTitle.ts` | `queryHaiku()` | Auto title |
| `tools/WebFetchTool/utils.ts` | `queryHaiku()` | Page summary |
| `tools/WebSearchTool/WebSearchTool.ts` | `queryModelWithStreaming()` | Search synthesis |
| `components/agents/generateAgent.ts` | `queryModelWithoutStreaming()` | Agent generation |
| `utils/hooks/skillImprovement.ts` | `queryModelWithoutStreaming()` | Skill hook eval |
| `commands/insights.ts` | `queryWithModel()` | Model-specific prompt |
| `services/PromptSuggestion/speculation.ts` | `queryModelWithStreaming()` | Suggestion speculation |
| `services/SessionMemory/sessionMemory.ts` | `queryHaiku()` | Memory extraction |

---

## 9. CCR (remote control) specifics

`isCCRMode()` = `isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)`
(`errors.ts:217-219`).

The API client itself does **not** fork for CCR — the SDK is still
standard first-party, just with an infrastructure-provided JWT in
place of user API key. What diverges:

- Auth errors (401/403) treated as transient, framed as network
  hiccups, retried immediately, even if server sends
  `x-should-retry: false` (`withRetry.ts:712-717`).
- User-facing message `CCR_AUTH_ERROR_MESSAGE` hides the "/login" hint
  since there's nothing the user can do.
- Transport-layer concerns (WebSocket vs SSE) live in `src/bridge/*`,
  not in this API layer.

---

## 10. Control-flow skeleton

```
caller (query.ts, etc.)
    ↓
queryModelWithStreaming() ─► queryModel()
    │                             ↓
    │      build system blocks, betas, tools (utils/api.ts)
    │                             ↓
    │      getAnthropicClient() [fresh per request]  ◄── providers
    │                             ↓
    │      withRetry({ ... }) ───► attempt loop:
    │                                  │
    │                                  ├── on 429/529 → backoff or fallback
    │                                  ├── on 401/403 → refresh + retry
    │                                  ├── on max-tokens → reduce & retry
    │                                  ├── on ECONNRESET → kill keep-alive
    │                                  └── on abort → APIUserAbortError
    │                             ↓
    │      beta.messages.stream(...)
    │                             ↓
    │      consume raw stream events (avoid O(n²) wrapper parse)
    │                             ↓
    │      updateUsage() + trackPromptCacheBreak() + logging
    ↓
StreamEvent / AssistantMessage / SystemAPIErrorMessage yields
```

---

## 11. Notable patterns and invariants

- **Client is never cached.** Every `getAnthropicClient()` call
  refreshes OAuth/AWS/GCP creds. Cheap because the SDK is a thin
  wrapper; expensive bits (GoogleAuth instance) are re-built but the
  auth token is cached inside it.
- **Raw stream events, not wrapper.** Parsing the high-level wrapper
  re-parses partial JSON per chunk — quadratic for large tool args.
  `claude.ts:1818-1819` has a comment marking the trade-off.
- **Cache preservation drives the fast-mode retry dual path.** The
  20-second threshold is tuned to the cache-TTL timing pattern: short
  waits keep the model variant stable (cache preserved), long waits
  switch variants (cache lost) to bound user wait time.
- **Thinking budget degrades gracefully** when max_tokens is reduced
  for context overflow — capped, not failed.
- **Persistent retry emits 30-s heartbeat yields** so host
  infrastructure doesn't mark unattended sessions idle. Callers should
  treat unexpected `SystemAPIErrorMessage` chunks mid-stream as
  keep-alive, not terminal.
- **Provider abstraction uses a type-cast lie** to present one
  `Anthropic` interface to downstream code. Only `messages.create()`
  / `beta.messages.stream()` are callable across all four.
- **CCR mode flips auth-error semantics** — 401/403 are retried, not
  surfaced as "log in."
- **Prompt-cache break detection has sticky-on invariants** (auto-mode,
  overage, cache editing) that act as regression canaries. If they
  appear in a `tengu_cache_break` event, something has un-latched
  them.
- **Haiku is excluded from cache-break tracking** to prevent false
  positives.

---

## 12. Flags for follow-up deep dives

The user-supplied "should flag" criterion on this task is *"if
retry/error recovery logic is complex."* It is. The following three
areas are each complex enough that a dedicated sub-analysis would pay
off:

### 12.1 `withRetry.ts` retry/fallback state machine ⚠️

822 LOC of branching — 4 provider-specific auth refresh paths, 2
fast-mode retry paths (short vs long wait), context-overflow token
adjustment, persistent-retry chunking with a separate counter,
Opus→Sonnet fallback gate, CCR 401/403 override, `x-should-retry`
header respect-with-exceptions, rate-limit-reset (`anthropic-
ratelimit-unified-reset`) vs `Retry-After` precedence, abort-signal
check at 3+ points. Worth a state-machine diagram + edge-case table.

### 12.2 Prompt-cache break detection (§5)

727 LOC, and the *sticky-on invariants* pattern is subtle — these are
regressions-as-events, not cache-misses. The interaction between
server TTL (5m / 1h) and client-side detection needs dedicated
treatment. Also, how tracked state interacts with subagent spawn/
despawn and the 10-source cap.

### 12.3 Error-classification taxonomy (`errors.ts`, 1,207 LOC)

~30+ error codes, ~20 user-facing message templates, reactive-compact
integration via `errorDetails`, media-size regex heuristics, CCR
reframing, SSL error extraction via `.cause` walking. Worth a
classification table with triggering HTTP status, code, message regex,
recovery action, and user-facing template.

### 12.4 Minor follow-ups

- OAuth token refresh under concurrent requests — race condition
  analysis of `handleOAuth401Error()` + `checkAndRefreshOAuthTokenIfNeeded()`.
- Bedrock/Vertex credential cache invalidation when underlying IAM
  role / service account key is genuinely expired (clearing cache
  doesn't help; what's the real recovery?).
- The 64k max-tokens threshold for streaming→non-streaming fallback:
  is this still the right cutoff given extended context?

---

## 13. Code references quick index

| Concern | File:line |
|---|---|
| Client entry | `src/services/api/client.ts:88-316` |
| Bedrock branch | `src/services/api/client.ts:153-189` |
| Vertex branch | `src/services/api/client.ts:221-298` |
| Foundry branch | `src/services/api/client.ts:191-219` |
| First-party branch | `src/services/api/client.ts:300-315` |
| Header injection | `src/services/api/client.ts:105-116, 330-354` |
| Request-ID injection | `src/services/api/client.ts:358-389` |
| `queryModel()` | `src/services/api/claude.ts:1017` |
| `queryModelWithStreaming()` | `src/services/api/claude.ts:752` |
| `queryModelWithoutStreaming()` | `src/services/api/claude.ts:709` |
| `queryHaiku()` | `src/services/api/claude.ts:3241` |
| `queryWithModel()` | `src/services/api/claude.ts:3300` |
| System prompt blocks | `src/services/api/claude.ts:1207-1264` |
| Thinking/effort/fast-mode | `src/services/api/claude.ts:1599-1645` |
| Streaming idle watchdog | `src/services/api/claude.ts:1868-1926` |
| Non-streaming fallback | `src/services/api/claude.ts:1768-1857` |
| Usage accumulation | `src/services/api/claude.ts:2924, 2993` |
| `withRetry` core loop | `src/services/api/withRetry.ts:96-548` |
| Context-overflow adjust | `src/services/api/withRetry.ts:388-427` |
| Fast-mode dual path | `src/services/api/withRetry.ts:267-314` |
| Opus→Sonnet fallback | `src/services/api/withRetry.ts:326-365` |
| Persistent retry | `src/services/api/withRetry.ts:96-104, 368-507` |
| CCR auth override | `src/services/api/withRetry.ts:712-717` |
| Error→message converter | `src/services/api/errors.ts:425-934` |
| Error classification | `src/services/api/errors.ts:965-1161` |
| Rate-limit headers | `src/services/api/errors.ts:465-558` |
| Media-size regex | `src/services/api/errors.ts:133-153` |
| Connection-error walk | `src/services/api/errorUtils.ts:42-83` |
| SSL code table | `src/services/api/errorUtils.ts:5-29` |
| HTML sanitization | `src/services/api/errorUtils.ts:122-130` |
| Nested message extraction | `src/services/api/errorUtils.ts:169-198` |
| Cache-break tracking | `src/services/api/promptCacheBreakDetection.ts:101-163` |
| Cache-break causes | `src/services/api/promptCacheBreakDetection.ts:28-99` |
| Logging events | `src/services/api/logging.ts:1-100` |
| `toolToAPISchema` | `src/utils/api.ts:119-160+` |
