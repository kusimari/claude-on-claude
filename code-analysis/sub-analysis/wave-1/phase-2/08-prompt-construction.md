# Prompt Construction for the Claude API

**Task**: #13 — "Analyze prompt building for Claude API"
**Owner**: analyst-1
**Wave / Phase**: 1 / 2
**Status**: Completed

---

## 1. Summary

The system prompt sent to the Claude API is *not* one monolithic string. It
is a **list of text blocks**, each potentially carrying its own
`cache_control` marker, built in two halves (static + dynamic) and then
re-partitioned by *content* (not position) at API send time into up to
four cache-scoped sub-blocks. The split exists so that the **global cache
scope** (cross-session, cross-org where authorized) can cover the static
half without being busted by session-specific runtime bits in the dynamic
half.

- Assembled in `src/constants/prompts.ts:getSystemPrompt` (and consumed
  via `src/utils/queryContext.ts:fetchSystemPromptParts`).
- Selected between default, custom, agent, coordinator, and override
  variants in `src/utils/systemPrompt.ts:buildEffectiveSystemPrompt`.
- A **boundary marker** (`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`) separates the
  two halves, at `prompts.ts:114`.
- `src/utils/api.ts:splitSysPromptPrefix` partitions the final array
  into `{attribution header, CLI prefix, static, dynamic}` blocks with
  explicit `cacheScope` per block.
- `src/services/api/claude.ts:buildSystemPromptBlocks` attaches
  `cache_control: {type:'ephemeral', scope?, ttl?}` onto the blocks.
- `addCacheBreakpoints` (`claude.ts:3063`) then places **exactly one
  message-level `cache_control` marker** on the last (or second-to-last
  for forks) message.
- Conversational context (`userContext`, `systemContext`) is attached
  separately — `userContext` is prepended as a user-role
  `<system-reminder>` message; `systemContext` is appended to the
  system prompt array.

**Caching is everything.** Every design decision visible in this
subsystem — ordering, boundary marker, split, one-marker invariant,
sticky beta-header latches, eligibility cached in bootstrap state —
exists to keep the server-side prompt cache intact across turns.
Violating any of them is measured in "~20K-70K tokens per flip".

---

## 2. The Five-Level Selection (`buildEffectiveSystemPrompt`)

`src/utils/systemPrompt.ts:41-123` picks *one* of five possible system
prompts per turn, in strict priority order:

```
0. override      — slash-command like /loop; REPLACES everything (appendSystemPrompt also skipped)
1. coordinator   — CLAUDE_CODE_COORDINATOR_MODE env-truthy; lazy-required to avoid cycle
2. agent         — mainThreadAgentDefinition.getSystemPrompt()
                    • Proactive/Kairos: APPENDED to default (agent adds domain on top)
                    • Otherwise: REPLACES default
3. custom        — --system-prompt CLI flag / SDK
4. default       — getSystemPrompt() (the 900-line registry in prompts.ts)
```

`appendSystemPrompt` is always added at the end *unless* an override was
selected. This is the contract QueryEngine wires up at
`QueryEngine.ts:286-325` — the final array is
`[...(custom ?? defaultSystemPrompt), ...memoryMechanics?, ...append?]`.

`buildEffectiveSystemPrompt` also logs `tengu_agent_memory_loaded` when
an agent definition opts in to memory (`systemPrompt.ts:86-97`) — this
is the main-thread counterpart to subagent memory tracking.

---

## 3. The Default System Prompt: Two-Half Structure

`src/constants/prompts.ts:getSystemPrompt` (lines 444-577) returns an
array shaped like:

```
Static (cacheScope='global' candidate)
  1.  getSimpleIntroSection(outputStyleConfig)       — identity + cyber-risk + URL rule
  2.  getSimpleSystemSection()                        — # System bullets (tools, hooks, reminders, compression)
  3.  getSimpleDoingTasksSection()                    — # Doing tasks (code style, user help)    [conditional]
  4.  getActionsSection()                             — # Executing actions with care
  5.  getUsingYourToolsSection(enabledTools)          — # Using your tools (Bash-vs-dedicated guidance)
  6.  getSimpleToneAndStyleSection()                  — # Tone and style
  7.  getOutputEfficiencySection()                    — # Communicating with the user / # Output efficiency

=== SYSTEM_PROMPT_DYNAMIC_BOUNDARY (only when shouldUseGlobalCacheScope()) ===

Dynamic (registry-managed via resolveSystemPromptSections)
  8.  session_guidance       — conditional: ask-user-question, shell !, agent/explore, skills, verifier
  9.  memory                 — loadMemoryPrompt()
 10.  ant_model_override     — ant-only defaultSystemPromptSuffix
 11.  env_info_simple        — # Environment (cwd, git, platform, shell, OS, model, cutoff)
 12.  language               — user language preference
 13.  output_style           — output style name + prompt
 14.  mcp_instructions       — UNCACHED: per-turn recompute; MCP may (re)connect
 15.  scratchpad             — if scratchpad enabled
 16.  frc                    — # Function Result Clearing (CACHED_MICROCOMPACT)
 17.  summarize_tool_results — "write down info from tool results"
 18.  numeric_length_anchors — ant-only: "≤25 words / ≤100 words"
 19.  token_budget           — TOKEN_BUDGET feature
 20.  brief                  — KAIROS/KAIROS_BRIEF: BRIEF_PROACTIVE_SECTION
```

The `CLAUDE_CODE_SIMPLE=1` early-return at `prompts.ts:450-454` bypasses
the entire assembly and returns only identity + cwd + date — used by
callers that want a minimal prompt for cheap classifier-style queries.

The **proactive/Kairos** path (`prompts.ts:466-489`) is a separate
assembly: it uses its own intro ("autonomous agent"), skips sections 3-7,
and appends `getProactiveSection()` at the end (the `<tick>`-based
pacing instructions, `prompts.ts:860-914`).

### 3.1 Why a boundary marker?

`prompts.ts:106-115` spells out the cache-scope contract:

> Everything BEFORE this marker in the system prompt array can use
> scope: 'global'. Everything AFTER contains user/session-specific
> content and should not be cached.

Global scope means "cacheable across sessions and (where authorized)
across orgs". Any session-specific bit in a `global`-scoped block
would either leak between sessions or invalidate the cache on every
turn. The boundary is a single sentinel string, and *two* downstream
pieces of code depend on it: `utils/api.ts:splitSysPromptPrefix` and
`services/api/claude.ts:buildSystemPromptBlocks` (comment at 110-113
names both).

### 3.2 systemPromptSection: memoization contract

`src/constants/systemPromptSections.ts:20-38` defines two factory
functions:

- `systemPromptSection(name, compute)` — cached until `/clear` or
  `/compact` (cacheBreak=false).
- `DANGEROUS_uncachedSystemPromptSection(name, compute, _reason)` —
  recomputed every turn, will cache-break on value change. Requires a
  `_reason` arg — not used at runtime, but forces the author to
  explain themselves. The only current uncached section is
  `mcp_instructions` (reason: "MCP servers connect/disconnect between
  turns").

`resolveSystemPromptSections` (lines 43-57) reads
`getSystemPromptSectionCache()` — a bootstrap-state-scoped map. The
cache is explicitly wiped on `/clear`/`/compact` by
`clearSystemPromptSections()` (lines 65-68), which also calls
`clearBetaHeaderLatches()` — the beta-header sticky-on mechanism
(§5.2) needs the same clearing semantics to keep the fresh conversation
reevaluating everything.

### 3.3 Why `mcp_instructions` is uncached

Comment at `prompts.ts:513-520`:

> When delta enabled, instructions are announced via persisted
> mcp_instructions_delta attachments instead of this per-turn
> recompute, which busts the prompt cache on late MCP connect.

There's an explicit migration path: when `isMcpInstructionsDeltaEnabled()`
is true, the section returns `null` and MCP instructions flow through
attachments on the user message instead, preserving the cache.

### 3.4 `token_budget` — a comment worth reading

`prompts.ts:540-549`:

> Cached unconditionally — the "When the user specifies..." phrasing
> makes it a no-op with no budget active. Was DANGEROUS_uncached
> (toggled on getCurrentTurnTokenBudget()), busting ~20K tokens per
> budget flip. Not moved to a tail attachment: first-response and
> budget-continuation paths don't see attachments (#21577).

This is the single clearest illustration of the mental model for
this file: make prompt strings **stateless narrators** ("when X
happens, do Y") rather than **state carriers** ("X is true, do Y"),
so they stay cacheable.

---

## 4. CLI Sysprompt Prefix + Attribution Header

`src/constants/system.ts` defines three identity-line prefixes:

```ts
const DEFAULT_PREFIX = "You are Claude Code, Anthropic's official CLI for Claude.";
const AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX =
  "You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.";
const AGENT_SDK_PREFIX =
  "You are a Claude agent, built on Anthropic's Claude Agent SDK.";
```

`CLI_SYSPROMPT_PREFIXES` (line 26) exposes them as a `ReadonlySet<string>`
so `splitSysPromptPrefix` can identify them **by content**, not by
position (needed because `getSimpleIntroSection` emits one of these at
the head of the prompt, and we need to detect it regardless of whether a
custom system prompt or override reshuffled everything).

`getCLISyspromptPrefix` (`system.ts:30-46`) picks:
- Vertex → DEFAULT (regardless of interactivity, for billing reasons)
- Non-interactive + hasAppendSystemPrompt → CLAUDE_CODE_PRESET (the
  "CLI, running within the SDK" variant)
- Non-interactive without append → AGENT_SDK
- Otherwise → DEFAULT

### 4.1 Attribution header

`getAttributionHeader(fingerprint)` at `system.ts:73-95` builds a
single-line string of the form:

```
x-anthropic-billing-header: cc_version=<VERSION>.<fingerprint>; cc_entrypoint=<entrypoint>;[ cch=00000;][ cc_workload=<w>;]
```

This is **prepended as a system prompt block** — not as an HTTP
header. It's identified downstream by `block.startsWith(
'x-anthropic-billing-header')`. Notable:

- `cc_version` embeds a *fingerprint* (computed from msg chars + version)
  so even same-version clients with different message content get
  different cache keys. Prevents accidentally sharing cache between two
  client builds with incompatible prompt payloads.
- `cch=00000` is a **placeholder for native attestation**. Comment:

  > Before the request is sent, Bun's native HTTP stack finds this
  > placeholder in the request body and overwrites the zeros with a
  > computed hash. The server verifies this token to confirm the
  > request came from a real Claude Code client.
  > (`system.ts:64-72`)

  Same-length replacement avoids Content-Length changes. Gated by
  `feature('NATIVE_CLIENT_ATTESTATION')`.
- `cc_workload` is a turn-scoped hint (`getWorkload()`) that lets the
  backend route e.g. cron-initiated requests to a lower QoS pool.
  Missing = interactive default. Safe re: fingerprint (fingerprint is
  computed before this line) and attestation (placeholder bytes are
  overwritten after).

### 4.2 Killswitch

`isAttributionHeaderEnabled` at `system.ts:52-57`:
- Env var `CLAUDE_CODE_ATTRIBUTION_HEADER=0` disables
- GrowthBook `tengu_attribution_header` default true (remote killswitch)

---

## 5. What the API Actually Sees

### 5.1 `splitSysPromptPrefix` — three modes

`src/utils/api.ts:321-435`. Input is the assembled `SystemPrompt`
(a string array with a brand). Output is `SystemPromptBlock[]`, each
with explicit `cacheScope`.

**Mode 1: MCP tools present** (`skipGlobalCacheForSystemPrompt=true`,
lines 326-360). Up to 3 blocks:
```
[attribution:null, CLI_prefix:'org', rest_concat:'org']
```
Comment (300-307): MCP tools carry a `cache_control` marker on the
tools array, and piling an extra system-prompt global-scope block on top
exceeds the "no more than 4 cache-control blocks per request" API limit.
Fall back to org caching on the system prompt.

**Mode 2: Global cache + boundary found** (lines 362-410). Up to 4
blocks:
```
[attribution:null, CLI_prefix:null, static_before_boundary:'global', dynamic_after:null]
```
This is the happy path: big shared-prefix segment cached at global scope,
cheap to hit across sessions.

**Mode 3: 3P providers or boundary missing** (lines 411-434). Same 3-block
shape as Mode 1 but with `cacheScope:'org'`.

Note `logEvent('tengu_sysprompt_missing_boundary_marker', …)` at 406-408
— catches the regression case where the boundary gets removed or the
assembly skips section 3-7.

### 5.2 `buildSystemPromptBlocks` — attach `cache_control`

`src/services/api/claude.ts:3213-3237`. Takes the split blocks and
attaches `cache_control: {type:'ephemeral', [scope], [ttl]}` where
`cacheScope !== null`. **Hard invariant** in the code:

```ts
// IMPORTANT: Do not add any more blocks for caching or you will get a 400
```

The API rejects requests with more than 4 cache-control blocks. The
function trusts `splitSysPromptPrefix` to have capped the count.

### 5.3 `getCacheControl` — ephemeral + scope + ttl

`claude.ts:358-374`. The returned object has:
- `type: 'ephemeral'` always (this is prompt caching, not the
  semantic-cache experiment).
- `ttl: '1h'` when `should1hCacheTTL(querySource)` returns true.
- `scope: 'global'` only when the caller requested it (the split
  function passes this for the static-block case).

`should1hCacheTTL` (`claude.ts:393-434`) cross-checks:
1. User eligibility: **ant**, **Bedrock + env opt-in**, or **claude.ai
   subscriber not currently on overage**.
2. Query-source allowlist: GrowthBook `tengu_prompt_cache_1h_config`
   with trailing-star patterns (`repl_main_thread*`, `agent:*`, `*`).

Both eligibility and allowlist are **latched in bootstrap state**
(`getPromptCache1hEligible/Allowlist`, set once). Comment at 403-405:

> Latch eligibility in bootstrap state for session stability — prevents
> mid-session overage flips from changing the cache_control TTL, which
> would bust the server-side prompt cache (~20K tokens per flip).

### 5.4 `addCacheBreakpoints` — exactly one message-level marker

`claude.ts:3063-3211`. Takes the whole conversation
`(UserMessage | AssistantMessage)[]` and produces `MessageParam[]` with:

- Exactly one `cache_control` block on the last message (normal) or
  second-to-last (for `skipCacheWrite` forks, e.g. fire-and-forget
  Agent tool calls). Dense reasoning at `claude.ts:3078-3088`:

  > Mycro's turn-to-turn eviction frees local-attention KV pages at any
  > cached prefix position NOT in `cache_store_int_token_boundaries`.
  > With two markers the second-to-last position is protected and its
  > locals survive an extra turn even though nothing will ever resume
  > from there — with one marker they're freed immediately. For
  > fire-and-forget forks we shift the marker to the second-to-last
  > message: that's the last shared-prefix point, so the write is a
  > no-op merge on mycro and the fork doesn't leave its own tail in the
  > KVCC.

- Marker lives on the last *block* of the last message (clone the
  content array first, line 622-629, to avoid mutating the source
  message — "splice contamination" across multiple addCacheBreakpoints
  calls on the same message).

- On assistant messages, the marker is NOT placed on `thinking` /
  `redacted_thinking` / `connector_text` blocks — those aren't stable
  enough to be good breakpoints (`claude.ts:658-661`).

- `useCachedMC` branch (CACHED_MICROCOMPACT) additionally manages
  `cache_edits` deletion blocks and `cache_reference` attachment on
  tool-result blocks that fall within the cached prefix. The edits
  are *pinned* (`pinCacheEdits`, line 3153) so re-sent at the same
  position on future calls — necessary because the API's
  cache-reference rule is "before or on the last cache_control", and
  shifting positions across turns would unpin stored deletions.

### 5.5 Per-message cache_control on the API request

Message-to-`MessageParam` translation:
`userMessageToMessageParam` (588-631), `assistantMessageToMessageParam`
(633-674). When `addCache=true`:
- String content → wrap in a text block with `cache_control`.
- Array content → attach `cache_control` to the last non-thinking block.
- Always clone the array first to avoid in-place mutation.

### 5.6 `userContext` ≠ system prompt

`userContext` (claudeMd + currentDate) is **not** in the system prompt
array. `src/utils/api.ts:449-474:prependUserContext` wraps it in a
`<system-reminder>` synthetic user message with `isMeta: true`:

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
<…>
# currentDate
Today's date is 2026-04-21.

      IMPORTANT: this context may or may not be relevant to your tasks. …
</system-reminder>
```

Tests get a no-op (`process.env.NODE_ENV === 'test'` at line 453) to
keep fixtures stable.

`systemContext` (gitStatus, optional cacheBreaker) **is** part of the
system prompt array via `appendSystemContext` (437-447): it
stringifies the object as `key: value\n…` and appends as a single
string block. Whether that string lands on the static or dynamic side
depends on whether it's inserted before or after the boundary — the
caller (QueryEngine) inserts it after the dynamic sections, so it's in
the dynamic side.

---

## 6. Sticky Beta-Header Latches

`claude.ts:1405-1456` shows another cache-preservation mechanism:

> Sticky-on latches for dynamic beta headers. Each header, once first
> sent, keeps being sent for the rest of the session so mid-session
> toggles don't change the server-side cache key and bust ~50-70K
> tokens.

Four latches currently:
- `afkModeHeaderLatched` — auto/AFK mode header (transcript classifier)
- `fastModeHeaderLatched` — `/fast` fast mode
- `cacheEditingHeaderLatched` — CACHED_MICROCOMPACT first-party only
- `thinkingClearLatched` — latches on first agentic query after 1h of
  idle (to force cache reset on dormancy)

All cleared by `clearBetaHeaderLatches()` at `/clear`/`/compact`
(hooked into `clearSystemPromptSections`). Per-call gates (e.g.
`isAgenticQuery`, `querySource === 'repl_main_thread'`) stay per-call so
non-agentic queries don't flip the main thread's state.

---

## 7. Tool Schemas Also Carry `cache_control`

`src/utils/api.ts:70-265` handles the *tool* side of the cache-key
equation. `BetaToolWithExtras.cache_control` (line 72-76) plus `strict`,
`defer_loading`, `scope`, `ttl` are the extensions beyond the base
Anthropic SDK type. The build picks exactly *one* tool to place the
marker on (handled upstream in the tool-registration pipeline, not
shown here). `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1` strips
everything except the base `cache_control` and standard fields
(`api.ts:234-260`).

Notable: `extraToolSchemas` like the `advisor_20260301` server tool
(`claude.ts:1386-1395`) are **appended after** the main tool array,
because the main array is what carries the `cache_control` marker:

> Server tools must be in the tools array by API contract. Appended
> after toolSchemas (which carries the cache_control marker) so
> toggling /advisor only churns the small suffix, not the cached
> prefix.

### 7.1 PROMPT_CACHE_BREAK_DETECTION

`claude.ts:1460-1486` shows the telemetry that watches the cache key
components. `recordPromptState` captures `{system, toolSchemas,
querySource, model, agentId, fastMode, globalCacheStrategy, betas,
autoModeActive, isUsingOverage, cachedMCEnabled, effortValue,
extraBodyParams}` — all the things the server hashes into the cache
key. The detector logs when any of these changes unexpectedly between
turns, giving an early warning on the regressions this subsystem exists
to prevent.

Notably excludes `defer_loading` tools from the hash (1461-1466): the
API strips them out, so including them would cause spurious
"tool schemas changed" breaks when tools are discovered or MCP servers
reconnect.

---

## 8. The Two Assembly Points

Two places call `getSystemPrompt()`:

### 8.1 `fetchSystemPromptParts`

`src/utils/queryContext.ts:44-74` — the common helper. Returns
`{defaultSystemPrompt, userContext, systemContext}`, all three fetched
in parallel.

Short-circuits when `customSystemPrompt !== undefined`:
`defaultSystemPrompt = []` and `systemContext = {}`. userContext still
runs (claudeMd still wanted even with a custom prompt). Comment at
`queryContext.ts:35-37` explains the logic: systemContext exists to
append to the default prompt, and with a custom prompt there's no
default to append to.

### 8.2 QueryEngine.ask

`src/QueryEngine.ts:286-325` — the caller that actually builds the
final array:

```ts
const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
]);
```

- `userContext` gets an extra merge with
  `getCoordinatorUserContext(mcpClients, scratchpadDir)` at
  QueryEngine.ts:302-308 — coordinator-mode injection.
- `memoryMechanicsPrompt` (lines 316-319) is a special case: when an
  SDK caller provides a custom prompt *and* has
  `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` set, prepend the memory-mechanics
  prompt. Explicit opt-in for external consumers.
- `asSystemPrompt` is a brand-enforcing wrapper (from
  `utils/systemPromptType.ts`) — prevents accidentally passing raw
  `string[]` into cache-sensitive functions.

The side-question fallback `buildSideQuestionFallbackParams` at
`queryContext.ts:88-179` is the second client; it rebuilds the same
shape when a mid-turn side question fires and there's no
`lastCacheSafeParams` snapshot yet. It mirrors the main assembly so the
rebuilt prefix matches — preserving the cache hit in the common case.
"May still miss the cache if the main loop applies extras this path
doesn't know about (coordinator mode, memory-mechanics prompt). That's
acceptable — the alternative is returning null and failing the side
question entirely." (83-87)

---

## 9. Data Flow End-to-End

```
┌─ setup() ─ bootstrap state, CLAUDE.md, memory, feature gates
│
└─ QueryEngine.ask (or print-mode)
   │
   ├─ fetchSystemPromptParts(tools, model, dirs, mcp, custom?)
   │    │   (queryContext.ts:44-74, Promise.all fan-out)
   │    ├─ getSystemPrompt(…)                        → string[] with boundary
   │    │    ├─ resolveSystemPromptSections(dynamic) → memoized sections
   │    │    └─ static sections (inline strings)
   │    ├─ getUserContext()                          → {claudeMd, currentDate}
   │    └─ getSystemContext()                        → {gitStatus?, cacheBreaker?}
   │
   ├─ buildEffectiveSystemPrompt({override, coord, agent, custom, default})
   │    → SystemPrompt (brand: string[])
   │
   ├─ appendSystemContext(sysprompt, systemContext)   — string stringified in-place
   │
   ├─ prependUserContext(messages, userContext)       — synthetic <system-reminder> user msg
   │
   └─ api.sendMessage(…)
      ├─ splitSysPromptPrefix(sysprompt)
      │    → [attribution:null, CLI_prefix:null, static:'global', dynamic:null]
      ├─ buildSystemPromptBlocks(...)                  — attaches cache_control
      ├─ addCacheBreakpoints(messages)                 — exactly one marker on last msg
      └─ POST /v1/messages { system: TextBlockParam[], messages, tools: [...], betas: [...], ... }
```

---

## 10. Prompt-Caching Optimization Notes — flagged per task description

The task asked to flag "if prompt caching or optimization needs analysis".
**Yes — prompt caching *is* the central design axis here.** This
subsystem is essentially one giant cache-preservation puzzle. Evidence:

1. **Three distinct cache-scope modes** depending on MCP / boundary /
   provider (§5.1).
2. **Exactly-one message-marker invariant** backed by an actual KVCC
   eviction explanation (§5.4).
3. **Sticky beta-header latches** (§6) — a separate class of
   cache-preservation mechanism that sits at the API-options level
   rather than the prompt level.
4. **Bootstrap-state latching** of eligibility + allowlist (§5.3) so
   that even mid-session state changes don't flip the TTL.
5. **Registry-with-explicit-uncached-escape-hatch** design for dynamic
   sections (§3.2) — the cacheability is the default, uncached is
   opt-in and requires a reason.
6. **`token_budget` refactor** from DANGEROUS_uncached to unconditional
   caching (§3.4) — a documented regression that cost ~20K tokens per
   budget flip.
7. **`logEvent` everywhere** (§5.1 `tengu_sysprompt_boundary_found` /
   `missing_boundary_marker`; §7.1 `PROMPT_CACHE_BREAK_DETECTION`) —
   continuous observability on the very failure modes the code is
   structured to prevent.

Further optimization opportunities worth a follow-up task:

- **Promote more dynamic sections to static**. `scratchpad`, `language`,
  `output_style`, `frc`, `summarize_tool_results`, `numeric_length_anchors`,
  `token_budget`, `brief` are mostly session-stable; several read from
  settings that change only on explicit user action. Moving them above
  the boundary would enlarge the global-cached block. The trade-off is
  cache-key fragmentation — if the setting varies per session, the
  global-cached segment fragments across the cacheScope='global' pool.
- **`mcp_instructions` delta migration** — comment at `prompts.ts:
  508-510` already flags this: when the delta attachments path is
  enabled, this section disappears and instructions flow via persisted
  attachments instead. This is a live migration; worth tracking
  adoption and deprecating the uncached path.
- **Attribution header in body** — currently the header is the first
  "system prompt block" and gets `cacheScope:null`, which is fine, but
  a comment about the header format notes that `cc_workload` and
  `cch` are tolerated as unknown fields by older servers. If the
  header ever becomes mandatory for billing correctness, the backward-
  compat path would need a plan. Not an immediate concern.

---

## 11. Key Data Structures

| Symbol | Location | Shape |
| --- | --- | --- |
| `SystemPrompt` | `utils/systemPromptType.ts` | Branded `string[]` |
| `SystemPromptSection` | `constants/systemPromptSections.ts:10-14` | `{name, compute, cacheBreak}` |
| `SystemPromptBlock` | `utils/api.ts:81-84` | `{text, cacheScope: 'org' \| 'global' \| null}` |
| `CacheScope` | `utils/api.ts` (inferred) | `'org' \| 'global'` |
| `TextBlockParam.cache_control` | SDK + `api.ts:70-76` extension | `{type:'ephemeral', scope?, ttl?:'5m'\|'1h'}` |
| `CachedMCEditsBlock` | `services/api/claude.ts:3052-3055` | `{type:'cache_edits', edits:[{type:'delete', cache_reference:string}]}` |
| `CachedMCPinnedEdits` | `claude.ts:3057-3060` | `{userMessageIndex, block}` |
| `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | `constants/prompts.ts:114` | Sentinel string |
| `CLI_SYSPROMPT_PREFIXES` | `constants/system.ts:26-28` | `ReadonlySet<string>` (3 entries) |

---

## 12. Integration Points

- **`bootstrap/state.ts`**: `getSystemPromptSectionCache`,
  `setSystemPromptSectionCacheEntry`, `clearSystemPromptSectionState`,
  `getPromptCache1hEligible/Allowlist`, `getAfkModeHeaderLatched`,
  `getFastModeHeaderLatched`, `getCacheEditingHeaderLatched`,
  `getThinkingClearLatched`, and their setters. All of these live in
  bootstrap state because they need cross-turn persistence but
  session-scoped lifetime.
- **`context.ts`**: memoized `getUserContext`, `getSystemContext`,
  `getGitStatus`; one-shot cache-break injection via
  `setSystemPromptInjection` (ant-only debugging).
- **`utils/claudemd.ts`**: `getClaudeMds`, `getMemoryFiles`,
  `filterInjectedMemoryFiles` — the CLAUDE.md loader chain.
- **`memdir/memdir.ts`**: `loadMemoryPrompt` for the `# memory` section.
- **`constants/outputStyles.ts`**: `getOutputStyleConfig` for the
  `# Output Style` section.
- **`services/analytics/growthbook.ts`**:
  `getFeatureValue_CACHED_MAY_BE_STALE` for 1h-cache allowlist and
  attribution killswitch.
- **`utils/betas.ts`**: `shouldUseGlobalCacheScope` — the single gate
  that enables Mode 2 in `splitSysPromptPrefix`.
- **`utils/model/providers.ts`**: `getAPIProvider` (`'firstParty'` /
  `'bedrock'` / `'vertex'`) — multiple cache decisions key off this.
- **`services/mcp/types.ts`**: `MCPServerConnection` — the
  `instructions` field feeds `getMcpInstructionsSection`.
- **`proactive/index.ts`**: `isProactiveActive` — branches the whole
  prompt assembly into the autonomous-agent path.
- **`tools/AgentTool/loadAgentsDir.ts`**: `AgentDefinition` — main-thread
  agent's `getSystemPrompt()` can replace or append.
- **`coordinator/coordinatorMode.ts`**: `getCoordinatorSystemPrompt`
  and `getCoordinatorUserContext` — lazy-required to break a cycle.

---

## 13. Historical Incidents Embedded in Comments

These are bugs or incidents the current structure exists to prevent —
worth preserving in a higher-level architectural note somewhere:

1. **#21577** (`prompts.ts:544-549`) — `token_budget` section was
   DANGEROUS_uncached, flipping on `getCurrentTurnTokenBudget()`, which
   busted ~20K tokens per budget flip. Fix: rephrase the prompt to be
   state-neutral so it can be unconditionally cached.
2. **PR #24490 / #24171** (`prompts.ts:346-348`) — session-variant
   guidance fragmenting the global-scope prefix hash into 2^N variants.
   Fix: segregate into `getSessionSpecificGuidanceSection` and place
   after the boundary.
3. **"~50-70K tokens per flip"** (`claude.ts:1406-1408`) — beta headers
   toggling mid-session. Fix: sticky-on latches.
4. **"~20K tokens per flip"** (`claude.ts:404-405`) — mid-session
   overage changing TTL from 1h to default. Fix: latch eligibility in
   bootstrap state.
5. **Splice contamination** (`claude.ts:622-624`) — multiple
   addCacheBreakpoints calls sharing the same array splice in duplicate
   cache_edits. Fix: clone array content.
6. **Mycro local-attention eviction** (`claude.ts:3078-3088`) — two
   cache-control markers per request caused dangling KV pages to
   survive an extra turn unnecessarily. Fix: exactly one marker.
7. **CACHED_MICROCOMPACT contamination across model types**
   (`claude.ts:3185-3187`) — mutating tool_result blocks in-place with
   `cache_reference` contaminated blocks reused by secondary queries on
   models without cache_editing support. Fix: create new objects, not
   mutate.
8. **PR #20357 resume tombstones** (`api.ts:702-713`) — old transcripts
   with synthetic FileEdit fields sending whole-file copies on resume.
   Fix: strip on `normalizeToolInputForAPI`.

---

## 14. Follow-up Recommendations

1. **Dedicated deep-dive on MCP instructions-delta attachment path**
   (`prompts.ts:508-520`, `attachments.ts:*`). This is a live
   migration from an uncached system-prompt section to a persisted
   attachment — understanding the attachment wire format and the
   per-server state machine would complete the picture.
2. **Dedicated deep-dive on CACHED_MICROCOMPACT**
   (`services/compact/cachedMCConfig.ts`,
   `claude.ts:3108-3162`). The cache_edits / cache_reference protocol
   is complex, has multiple "pinned at original position" invariants,
   and has already had at least one contamination bug. A separate
   report detailing the deletion-sequencing rules would be valuable.
3. **Map which dynamic sections are actually used per typical
   session**. Some (`brief`, `token_budget`, proactive sections) are
   feature-gated and rare. Understanding the common case helps target
   optimization at the sections that actually affect cache rates.
4. **Trace `getWorkload()` back to its setters** (cron? template
   jobs? Agent SDK?). The `cc_workload` attribution field is the
   only runtime input to the attribution header besides fingerprint
   and entrypoint — worth understanding what sets it and when.
5. **Cross-reference with task #15 (Claude API communication
   patterns) and #12 (API response handling)**. The cache-control
   tagging work here connects directly to how responses report
   `cache_creation_input_tokens` / `cache_read_input_tokens` and how
   those feed back into the loop's token-budget and
   cost-tracking logic.
6. **Cross-reference with task #17 (query.ts orchestration)**. The
   boundary-marker split and the sticky-header latches are things
   query.ts' continuation paths must respect — re-entry into
   `queryLoop()` at line 1715-1727 needs to produce a cache-compatible
   system prompt, not rebuild from scratch.

---

## 15. Inventory of Files Read

| File | Purpose |
| --- | --- |
| `src/constants/prompts.ts` (0-915) | The 900-line default system prompt registry |
| `src/constants/systemPromptSections.ts` (full) | Memoized-section factory + cache policy |
| `src/constants/system.ts` (full) | CLI prefix + attribution header |
| `src/utils/systemPrompt.ts` (full) | 5-level prompt selection |
| `src/utils/queryContext.ts` (full) | `fetchSystemPromptParts` + side-question fallback |
| `src/context.ts` (full) | `getUserContext`, `getSystemContext`, `getGitStatus` |
| `src/utils/api.ts:30, 70-475` | `splitSysPromptPrefix`, `appendSystemContext`, `prependUserContext`, tool-schema cache-control |
| `src/services/api/claude.ts:358-435` | `getCacheControl` + `should1hCacheTTL` |
| `src/services/api/claude.ts:588-674` | message→MessageParam translation |
| `src/services/api/claude.ts:1370-1486` | top-level assembly at API boundary + beta-header latches + cache-break detection |
| `src/services/api/claude.ts:3050-3237` | `addCacheBreakpoints` + `buildSystemPromptBlocks` |
| `src/QueryEngine.ts:280-400` | assembly caller — merges memory-mechanics + coordinator userContext |
