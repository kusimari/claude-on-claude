# 03 — Tool System Init: Loading, Registration, and Permissions

**Scope**: How every tool the agent can call gets (a) defined, (b) listed,
(c) merged with MCP and permission-context state, (d) filtered by mode /
agent / simple-mode, (e) serialized for the Anthropic API, and (f) gated
at invocation time by the permission pipeline. The tour starts at the
`buildTool()` factory in `src/Tool.ts`, walks through `src/tools.ts` and
`src/utils/toolPool.ts`, and ends at the 8-step check in
`src/utils/permissions/permissions.ts`.

---

## 1. Tool registry: the `Tool<Input, Output, P>` shape

### 1.1 `buildTool()` and `TOOL_DEFAULTS`

Every tool is constructed through `buildTool(def)` at `src/Tool.ts:783`.
The factory merges a partial `ToolDef<...>` on top of `TOOL_DEFAULTS`
(`Tool.ts:757-780`), so the call site only has to specify fields that
differ from the defaults. Defaults worth remembering:

| Field | Default |
|---|---|
| `isEnabled` | `() => true` |
| `isConcurrencySafe` | `() => false` |
| `isReadOnly` | `() => false` |
| `isDestructive` | `() => false` |
| `checkPermissions` | passthrough "allow" |
| `toAutoClassifierInput` | `() => ''` |
| `userFacingName` | `() => def.name` |

The defaults are intentionally conservative: a newly written tool is
serial, write-capable, and cleared to run. The surface of `Tool` itself
(`Tool.ts:380-690`) is ~40 fields: `call()`, `description()`,
`inputSchema` (Zod) and `inputJSONSchema` (raw JSON Schema for MCP),
`checkPermissions()`, `inputsEquivalent()`, `renderToolUseMessage()`,
`prompt()`, `mapToolResultToToolResultBlockParam()`,
`backfillObservableInput()`, `validateInput()`, `getPath()`,
`preparePermissionMatcher()`, `interruptBehavior`,
`isSearchOrReadCommand`, `shouldDefer`, `alwaysLoad`, `mcpInfo`, etc.
Each field has a specific stage of the agent loop that reads it:

- `description()` + `prompt()` → serialized into system prompt and
  the `tools` block of every request.
- `inputSchema` (Zod) or `inputJSONSchema` (MCP) → converted by
  `getToolForApi` for the API body.
- `checkPermissions` → the 1c step of the permission pipeline.
- `preparePermissionMatcher` → normalizes the "rule input" (e.g.,
  `Bash("ls *")` → `"ls *"`) used by permission rule matching.
- `validateInput` + `backfillObservableInput` → pre-call normalization
  and post-call observability/replay normalization.
- `isReadOnly`, `isConcurrencySafe` → orchestrator concurrency policy
  in `query.ts` (governs which tool calls can run in parallel).
- `isDestructive` → rendering hints and acceptEdits gating.
- `interruptBehavior` → how cancellation propagates into a long call.
- `shouldDefer` → whether a tool needs deferred execution (used by
  EnterPlanMode/ExitPlanMode/ScheduleWakeup kinds of "transfer of
  control" tools).

The shape is mechanical; the interesting business logic is in the
individual `tools/*/...` folders and in how these are selected.

### 1.2 Empty permission context

`getEmptyToolPermissionContext()` at `Tool.ts:140` is the canonical
zero-state used in scripts, tests, and headless daemons that need to
call tools without a REPL: default mode, no rules, no working
directories, no bypass availability. Every real session starts by
constructing one of these and then transforming it via
`initializeToolPermissionContext()`.

---

## 2. Tool assembly pipeline

The assembly pipeline takes a `ToolPermissionContext` and a set of MCP
tools from the MCP router, and emits the final `Tools` array that the
API request uses. The pipeline has five stages:

```
getAllBaseTools()
   ↓
getTools(permissionContext)     ← simple-mode / deny / REPL / isEnabled filters
   ↓
assembleToolPool(context, mcpTools)  ← + MCP, dedup, partition-sort
   ↓
useMergedTools hook             ← + initialTools, coordinator filter
   ↓
getToolForApi(tool, context)    ← JSONSchema serialization (cached)
```

### 2.1 `getAllBaseTools()` (`src/tools.ts:193`)

The canonical, ordered list of every built-in tool. Ordering matters:
the first chunk of the final tool array is the contiguous "built-in
prefix" for prompt-cache stability (see §2.4). The function uses
*conditional inserts* gated on `feature('FLAG')` and `USER_TYPE ===
'ant'` to keep bundle size small and let dead-code elimination delete
unused tool modules:

```ts
if (feature('AGENT_TOOL') && USER_TYPE === 'ant') {
  tools.push(AgentTool)
}
```

Patterns to watch for:

- **Plain `require` inside an `if (feature('X'))`**: the require is
  hoisted by Bun only when the feature is on, letting tree-shaking
  drop the entire module otherwise.
- **`USER_TYPE === 'ant'`**: Anthropic-only tools (AgentTool,
  CronCreate, TeamCreate, etc.).
- **Always-present tools**: Bash, Edit, Read, Write, Glob, Grep, LS,
  NotebookEdit, TodoWrite, WebFetch, WebSearch, MemoryTool.

### 2.2 `getTools(permissionContext)` (`src/tools.ts:271`)

Layers four filters over `getAllBaseTools()`:

1. **Simple mode carve-out**: `process.env.CLAUDE_CODE_SIMPLE === '1'`
   trims the set to just `[Bash, Read, Edit]` and returns early.
2. **Special-tool exclusion**: `ListMcp`, `ReadMcp`, and
   `SyntheticOutputTool` are filtered out of the default list — they
   are added later by callers that need them (MCP inspector,
   synthetic-output test harness).
3. **REPL-mode filter**: when the `REPLTool` is in the pool,
   `REPL_ONLY_TOOLS` are hidden because the REPL tool provides its
   own variants inside the sandbox. The filter key is `t.name`.
4. **Deny-rule filter** (`filterToolsByDenyRules`, `tools.ts:262`):
   if the permission context's `alwaysDeny` rules name a *tool
   itself* (not a specific input), the tool is stripped from the
   catalog entirely so the model never sees it. Input-scoped denials
   are left alone — they're handled by the permission pipeline at
   call time.
5. **isEnabled filter**: each tool's `isEnabled()` is awaited; false
   removes it.

### 2.3 `assembleToolPool(context, mcpTools)` (`src/tools.ts:345`)

Combines the base tools with the MCP tools produced by the MCP router.
`uniqBy(..., 'name')` deduplicates using the first occurrence — base
tools precede MCP, so if an MCP server happens to register a tool
with a colliding name the base tool wins.

The array is then partitioned by `isMcpTool` and each partition is
sorted alphabetically:

```ts
const [mcp, builtIn] = partition(merged, isMcpTool)
const byName = (a, b) => a.name.localeCompare(b.name)
return [...builtIn.sort(byName), ...mcp.sort(byName)]
```

This shape — a contiguous built-in prefix followed by MCP tools — is
load-bearing. The Anthropic API's prompt-cache bucket is keyed on a
prefix of the tool list; changing the order of built-ins between
requests would bust the cache. Anything downstream that reorders tools
has to preserve this partition (hence `mergeAndFilterTools` in §2.4
does the same partition).

### 2.4 `mergeAndFilterTools()` (`src/utils/toolPool.ts:55`)

Pure, React-free function. Called from the `useMergedTools` hook
(`src/hooks/useMergedTools.ts`) inside a `useMemo`. Given:

- `initialTools` — extras passed as a prop (e.g., startup MCP that
  shipped with the SDK invocation),
- `assembled` — the output of `assembleToolPool`,
- `mode` — the current permission mode,

it:

1. Concatenates `[...initialTools, ...assembled]`; `uniqBy('name')`
   drops duplicates, giving `initialTools` precedence. This is
   specifically how SDK callers override a built-in with their own
   variant.
2. Re-applies the partition-sort (§2.3) so ordering still matches
   the cache policy even after the prepend.
3. If `feature('COORDINATOR_MODE')` is on and
   `coordinatorModeModule.isCoordinatorMode()`, narrows the pool to
   `COORDINATOR_MODE_ALLOWED_TOOLS` plus the PR-activity
   subscription tools (suffix match — the MCP server prefix varies).

Why a separate file from `useMergedTools`? `print.ts` (the headless
SDK/print entrypoint) needs the same logic but must not import
React/Ink. Keeping this in `utils/toolPool.ts` leaves the React hook
as a 45-line thin wrapper.

### 2.5 `getToolForApi()` — schema serialization

Final stage (`src/utils/api.ts:135`). Converts a `Tool` into the
Anthropic API `ToolUnion` shape for the request body. Key behaviors:

- If `tool.inputJSONSchema` is present (always for MCP tools) it's
  used verbatim — skips `zodToJsonSchema` conversion. MCP servers
  ship their own schemas and we don't want to round-trip them.
- Otherwise `zodToJsonSchema(tool.inputSchema)` is called and the
  result is cached. The cache is keyed on the serialized output so
  repeated tools share the same object (helps string interning).
- Strict mode is gated on the `tengu_tool_pear` statsig experiment.
- `eager_input_streaming: true` is attached when the tool's
  streaming-input flag is set.
- Swarm-only fields (`agentType`, etc.) are stripped so they don't
  leak into the API call.

---

## 3. Permission context initialization

### 3.1 `initializeToolPermissionContext` (`permissionSetup.ts:872`)

This is the one-time construction of the `ToolPermissionContext` at
session start. It happens *after* `setup()` has committed the cwd and
hook snapshot (see `01-setup-environment.md`). Inputs: parsed CLI
flags, `settings.json` (user + project + enterprise + policy),
bootstrapped user trust state, and the current `allowedTools`/
`disallowedTools` CLI arrays.

Key steps:

1. **CLI tool parsing**: `--allowedTools "Bash,Edit"` becomes a set of
   `alwaysAllow` rules at CLI source; likewise for `--disallowedTools`.
   Each string is parsed into a rule with the source tagged as `cliArg`
   so it can be distinguished from settings-derived rules.
2. **Base-tool deny expansion**: settings can specify wildcards
   (`Bash(*)`) that are expanded into the rule set with a "source"
   attribution so later operations like `deletePermissionRule` know
   where to edit.
3. **Additional working directories**: `--add-dir` paths and
   `settings.permissions.additionalDirectories` are normalized to
   absolute paths and deduplicated into a `Map` keyed on absolute
   path, with the value being the rule source.
4. **Mode selection**: `initialPermissionModeFromCLI()`
   (`permissionSetup.ts:689`) applies priority:
   `--dangerously-skip-permissions` > CLI `--permission-mode` >
   `settings.defaultMode`. If `CCR_ACTIVE === '1'` (a trusted
   launcher), the CCR override is applied afterward.
5. **Bypass availability gate**: `isBypassPermissionsModeAvailable` is
   computed from settings (`disableBypassPermissionsMode`) and
   enterprise policy; if disabled the mode cannot be *entered* later.
6. **Auto-mode availability gate**: `isAutoModeAvailable` comes from
   feature flags, user type, and settings.
7. **Dangerous-permission handling for auto mode**: If the initial
   mode is `auto`, `stripDangerousPermissionsForAutoMode` is called,
   moving rules like `Bash(*)`, `Bash(python:*)`, etc. into
   `strippedDangerousRules` so they can be restored when the user
   exits auto mode. `isDangerousBashPermission` /
   `isDangerousPowerShellPermission` / `isDangerousTaskPermission`
   (`permissionSetup.ts:94-245`) are the predicates.
8. **Plan-mode preparation**: `prepareContextForPlanMode` stashes
   `prePlanMode` so we can restore state on ExitPlanMode.

The final `ToolPermissionContext` has shape:

```ts
{
  mode: 'default' | 'plan' | 'acceptEdits' | 'bypassPermissions'
       | 'auto' | 'dontAsk',
  additionalWorkingDirectories: Map<string, RuleSource>,
  alwaysAllow: RulesBySource,
  alwaysDeny: RulesBySource,
  alwaysAsk: RulesBySource,
  isBypassPermissionsModeAvailable: boolean,
  isAutoModeAvailable: boolean,
  strippedDangerousRules?: RulesBySource,  // auto mode only
  shouldAvoidPermissionPrompts: boolean,
  awaitAutomatedChecksBeforeDialog: boolean,
  prePlanMode?: PermissionMode,
}
```

### 3.2 Mode transitions (`transitionPermissionMode`)

`permissionSetup.ts:597` is the single choke point for mode changes
mid-session (via keyboard shortcut, Shift+Tab, `/mode`, or
ExitPlanMode). It enforces the availability gates, performs
`strip`/`restore` of dangerous permissions across auto-mode entry/exit,
invokes `transitionPlanAutoMode` for plan→auto and auto→plan
transitions, and publishes the new mode via `setAppState`.

### 3.3 Async auto-mode gate (`verifyAutoModeGateAccess`)

`permissionSetup.ts:1078` performs a *fresh* GrowthBook config fetch
at the moment of entering auto mode (not at startup). The returned
config includes a *transform function* that is run against the live
`ToolPermissionContext` — this is how server-side policy can tighten
the rule set for specific users without requiring a new release.
Failure mode: if the fetch fails or the transform throws, we fall
back to a denial path with `tengu_iron_gate_closed` fail-closed
telemetry.

---

## 4. 8-step permission pipeline

`hasPermissionsToUseToolInner()` at `permissions.ts:1158` is the
core rule evaluator. Its outer wrapper `hasPermissionsToUseTool()`
at `permissions.ts:473` adds auto-mode classification,
`dontAsk`-mode short-circuit, and PermissionRequest hook emission
for headless agents.

The 8 steps, in order:

### Step 1a — `alwaysDeny` hard stop

If any deny rule matches (via `toolMatchesRule`, `permissions.ts:238`),
return `{ behavior: 'deny', decisionReason: { type: 'rule', rule } }`
immediately. Deny wins over everything.

### Step 1b — `alwaysAsk` forced-ask

If an ask rule matches, skip to the prompt branch regardless of
later allow rules. Intent: a user who explicitly opted into "ask
me about X" should never be surprised by a later blanket allow.

### Step 1c — `tool.checkPermissions`

Each tool's own `checkPermissions()` is called (default is passthrough
"allow"). Tools like Bash run their own command-shape validation
here (e.g., classifying commands with arbitrary flag combinations
as "ask"). Can return any of `allow | ask | deny | passthrough`.

### Step 1d — Tool-level deny propagation

If `checkPermissions` returned `deny`, we return deny directly — the
tool's own safety check wins over mode.

### Step 1e — `requiresUserInteraction`

Some tools force a prompt regardless of rules (e.g., tools that write
to a shared external system). These short-circuit to `ask`.

### Step 1f — Content-specific ask

For tools like Edit, the rule may be scoped to a *file* (`Edit(src/foo.ts)`),
so the check compares against the specific input. A matching
content-ask rule forces the prompt.

### Step 1g — `safetyCheck`

A final catch-all that can be bypass-immune (see Step 2a). Used for
never-bypassable invariants (e.g., preventing a WriteFile call from
escaping the project root even in bypass mode).

### Step 2a — `bypassPermissions` mode

If the mode is `bypassPermissions` *and* none of the bypass-immune
safety checks fired, return allow. This is the "YOLO mode" short
circuit, but it's deliberately placed *after* safetyChecks so the
bypass cannot override a hard-coded guardrail.

### Step 2b — `alwaysAllow` rule match

If an allow rule matches, return allow with the rule as the decision
reason. Tracked for telemetry.

### Step 2c — Passthrough → Ask

If nothing matched, the default is to ask the user. For auto mode
this is where `TRANSCRIPT_CLASSIFIER` steps in (§6) to convert an
ask into an allow based on model output.

### Outer wrapper specifics (`hasPermissionsToUseTool`)

- Auto-mode classifier pre-pass: if the mode is `auto` and the tool
  is in the classifier allowlist, a speculative classifier call is
  kicked off before the inner permission check. A 2-second grace
  period lets the classifier return and short-circuit an `ask`
  result into `allow` before the interactive prompt appears.
- Consecutive-denial tracking: `persistDenialState` bumps a counter
  per tool; once `DENIAL_LIMITS` is hit, auto mode forces the user
  to acknowledge.
- PermissionRequest hook: headless agents (async subagents,
  coordinator with remote workers) emit a hook event so the
  coordinator can approve/deny on behalf of the worker.
- `dontAsk` mode: silently denies any result that would have been
  `ask`. Used by the `--dont-ask` scripted-invocation flag.

---

## 5. Permission modes summary

| Mode | Pipeline behavior | Notes |
|---|---|---|
| `default` | Full 8-step pipeline | The normal interactive mode. |
| `plan` | `checkPermissions` may return `deny` for tools not in the plan whitelist; everything else asks. | ExitPlanMode is always allowed. `prePlanMode` stashes the prior mode for restoration. |
| `acceptEdits` | Destructive=false + readonly tools auto-allowed; Edit/Write skip ask. | Still respects deny rules. |
| `bypassPermissions` | Skips steps 1a-1g (except bypass-immune safetyChecks) and returns allow. | Gated by `isBypassPermissionsModeAvailable` + root check + sandbox requirement for ant users (`setup.ts:396-442`). |
| `auto` | Classifier converts `ask` → `allow`; dangerous permissions stripped. | Requires `isAutoModeAvailable` + GrowthBook transform. |
| `dontAsk` | `ask` results silently become `deny`. | Scripted/noninteractive use. |

---

## 6. Auto-mode classifier

Auto mode (`permissions.ts:473-...` wrapper) layers a transcript-based
classifier on top of the rule pipeline.

### 6.1 Dangerous-permission stripping

Entering auto mode calls `stripDangerousPermissionsForAutoMode`
(`permissionSetup.ts:510`). The predicates:

- `isDangerousBashPermission`: removes `Bash(*)`, `Bash(python:*)`,
  `Bash(node:*)`, `Bash(sh:*)`, and other permissive Bash rules
  that would grant arbitrary command execution.
- `isDangerousPowerShellPermission`: equivalent for PowerShell.
- `isDangerousTaskPermission`: equivalent for the `Task` subagent
  tool (prevents a subagent from inheriting arbitrary-command
  authority in auto mode).

Stripped rules are preserved in `strippedDangerousRules` so
`restoreDangerousPermissions` (`permissionSetup.ts:561`) can put
them back on exit.

### 6.2 Classifier decisioning

The classifier receives a redacted transcript plus the pending tool
input and returns `allow | ask | deny`. The outer wrapper uses this
to promote an `ask` result from the inner pipeline into `allow`.

### 6.3 Circuit breaker

`tengu_auto_mode_config.enabled` is re-checked per decision — a
server-side kill switch. If flipped off, auto mode silently reverts
to default-mode behavior (each tool call prompts).

### 6.4 Denial tracking

`persistDenialState` tracks:

- `DENIAL_LIMITS.consecutive` — consecutive denials of the *same*
  tool.
- `DENIAL_LIMITS.total` — across the whole session.

Exceeding either forces a user-acknowledgment prompt and can trigger
`localDenialTracking` for async subagents (where the user isn't
attached to the worker REPL).

### 6.5 Speculative Bash classifier race

`useCanUseTool.tsx` (around line 60-90 in the compiled file) starts
a `BASH_CLASSIFIER` request concurrently with the inner permission
check when the incoming tool is `Bash` and auto mode is active. If
the classifier resolves within a 2s grace window, its decision is
applied. Otherwise the interactive prompt wins. This is the
"classifier-pre-approval" pattern that keeps auto mode feeling
responsive even when the classifier is slow.

---

## 7. Agent-specific tool filtering

Subagents (Task-tool spawns, Agent SDK) see a *different* tool pool
than the main thread. The filtering is centralized in
`tools/AgentTool/agentToolUtils.ts`.

### 7.1 `filterToolsForAgent(tools, agentDef, isAsync, isBuiltIn)`
(`agentToolUtils.ts:70`)

Filter layers:

1. `ALL_AGENT_DISALLOWED_TOOLS` (`constants/tools.ts`) — tools that
   no subagent should ever have. Includes things like AgentTool
   (no recursive spawning), EnterPlanMode, TaskCreate/TaskUpdate
   (only the coordinator manages tasks).
2. `CUSTOM_AGENT_DISALLOWED_TOOLS` — additional tools hidden from
   user-defined custom agents but still visible to built-in agents.
   `isBuiltIn` skips this layer.
3. `ASYNC_AGENT_ALLOWED_TOOLS` — for async subagents (spawned via
   PR review, SDK), only this allowlist is exposed. MCP tools are
   deliberately excluded (see comment at `constants/tools.ts:91`).
4. ExitPlanMode carve-out: if the main thread is in plan mode, the
   subagent can still call ExitPlanMode to propagate the exit.
5. `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` carve-outs for the swarm
   teammate mode — includes orchestration tools the teammate
   needs (SendMessage, TaskCreate, etc.).

### 7.2 `resolveAgentTools(...)` (`agentToolUtils.ts:122`)

Driven by the agent definition's `tools` field:

- `'*'` wildcard expands to "all tools permitted by
  `filterToolsForAgent`".
- Explicit names select a subset; names not present in the base
  pool are logged as misconfigurations.
- `Agent(agentType)` syntax in the definition is parsed into
  metadata that flows into `filterDeniedAgents`
  (`permissions.ts:325`) so a subagent can be prevented from
  spawning *other* specific agent types.
- **Main-thread skip**: when `isMain === true`, we skip
  `filterToolsForAgent` entirely and return the full pool — the
  main thread is the coordinator and sees everything.

### 7.3 Permission context passthrough

Subagents inherit the parent's `ToolPermissionContext` with a few
transformations:

- `additionalWorkingDirectories` is passed through verbatim (the
  subagent is scoped to the same filesystem).
- Mode is inherited — a subagent in plan mode stays in plan.
- `awaitAutomatedChecksBeforeDialog` is force-true so async
  subagents never block on an interactive prompt.
- `shouldAvoidPermissionPrompts` is force-true for async subagents;
  a denied `ask` becomes an implicit `deny` and the subagent
  reports failure.

---

## 8. Integration points and follow-up deep dives

### 8.1 Where tools flow through

| Caller | Entry | Output used for |
|---|---|---|
| Main REPL | `useMergedTools` hook | Rendered tool picker + API tools block |
| Print/SDK headless | `mergeAndFilterTools` direct call | API tools block (no React) |
| Subagent (Task tool) | `resolveAgentTools` | Subagent-local tool pool |
| API request assembly | `getToolForApi` per tool | JSON-Schema tool definitions in request body |
| Permission check | `hasPermissionsToUseTool` | Gate at call time |
| MCP inspector | `getTools` + `ListMcp`/`ReadMcp` injected | Inspector UI only |

### 8.2 Code references quick index

| Concern | File:line |
|---|---|
| `Tool` type definition | `src/Tool.ts:380-690` |
| `TOOL_DEFAULTS` | `src/Tool.ts:757-780` |
| `buildTool()` | `src/Tool.ts:783` |
| `getEmptyToolPermissionContext` | `src/Tool.ts:140` |
| `getAllBaseTools` | `src/tools.ts:193` |
| `getTools` | `src/tools.ts:271` |
| `filterToolsByDenyRules` | `src/tools.ts:262` |
| `assembleToolPool` | `src/tools.ts:345` |
| `getMergedTools` | `src/tools.ts:383` |
| `mergeAndFilterTools` | `src/utils/toolPool.ts:55` |
| `applyCoordinatorToolFilter` | `src/utils/toolPool.ts:35` |
| `useMergedTools` hook | `src/hooks/useMergedTools.ts` |
| `initSinks` (tangent) | `src/utils/sinks.ts:13` |
| Tool group constants | `src/constants/tools.ts` |
| `hasPermissionsToUseTool` outer | `src/utils/permissions/permissions.ts:473` |
| `hasPermissionsToUseToolInner` | `src/utils/permissions/permissions.ts:1158` |
| `checkRuleBasedPermissions` | `src/utils/permissions/permissions.ts:1071` |
| `toolMatchesRule` | `src/utils/permissions/permissions.ts:238` |
| `filterDeniedAgents` | `src/utils/permissions/permissions.ts:325` |
| `initializeToolPermissionContext` | `src/utils/permissions/permissionSetup.ts:872` |
| `initialPermissionModeFromCLI` | `src/utils/permissions/permissionSetup.ts:689` |
| `isDangerousBashPermission` | `src/utils/permissions/permissionSetup.ts:94` |
| `stripDangerousPermissionsForAutoMode` | `src/utils/permissions/permissionSetup.ts:510` |
| `restoreDangerousPermissions` | `src/utils/permissions/permissionSetup.ts:561` |
| `transitionPermissionMode` | `src/utils/permissions/permissionSetup.ts:597` |
| `verifyAutoModeGateAccess` | `src/utils/permissions/permissionSetup.ts:1078` |
| `prepareContextForPlanMode` | `src/utils/permissions/permissionSetup.ts:1462` |
| `transitionPlanAutoMode` | `src/utils/permissions/permissionSetup.ts:1502` |
| `useCanUseTool` hook | `src/hooks/useCanUseTool.tsx` |
| `filterToolsForAgent` | `src/tools/AgentTool/agentToolUtils.ts:70` |
| `resolveAgentTools` | `src/tools/AgentTool/agentToolUtils.ts:122` |
| `getToolForApi` | `src/utils/api.ts:135` |

### 8.3 Subsystems flagged for dedicated analysis

Per the team-lead directive, this report should flag which tool
categories warrant their own dedicated reports. Findings:

1. **Bash tool deep-dive (RECOMMENDED)**. The Bash tool is not a thin
   wrapper — it owns command classification, the
   `BASH_CLASSIFIER` speculative approval race, shell-escape
   handling, timeout policy, and the `preparePermissionMatcher`
   logic that normalizes command strings to matchable rules. The
   `isDangerousBashPermission` predicate and its auto-mode
   stripping are Bash-specific policy. This is a substantial
   subsystem on its own.
2. **MCP tool generation + router (RECOMMENDED)**. How MCP servers
   are discovered, how `mcpInfo` is attached, how
   `inputJSONSchema` flows from the server through
   `assembleToolPool` to `getToolForApi`, how
   `mcp__server__tool` naming conventions and `mcp__server`
   wildcard rules interact with `toolMatchesRule`. The MCP
   subsystem is ~20 files and is the main "extensibility" vector.
3. **Agent / Task tool deep-dive (RECOMMENDED)**. The AgentTool
   implementation (`tools/AgentTool/*.ts`) spans ~2k LOC:
   definition resolution, subagent lifecycle, streaming back into
   the coordinator, `filterToolsForAgent` / `resolveAgentTools`,
   in-process vs. out-of-process teammate modes, and the
   `Agent(agentType)` rule syntax. Only the tool-filtering edges
   are covered here.
4. **Coordinator mode deep-dive (worth it)**. `coordinatorMode.ts`
   + `COORDINATOR_MODE_ALLOWED_TOOLS` + PR-activity subscription
   + the swarm backend interactions. The integration with
   `mergeAndFilterTools` is simple, but the mode's state machine
   and interaction with the Task system is non-trivial.
5. **MemoryTool / TodoWrite / Memory system**. These tools own
   state that survives the session (user memory files, todo
   persistence). Not complex individually but the memory
   file format, precedence, and loader should be one analysis.
6. **Permission rules file format + persistence**. The
   `RulesBySource` shape, the settings.json schema for
   `permissions.allow`/`deny`/`ask`, how
   `applyPermissionRulesToPermissionContext` /
   `syncPermissionRulesFromDisk` / `deletePermissionRule`
   interact with the on-disk settings stack (user, project,
   enterprise, policy), and the rule-source precedence rules.
   Covered here only in summary.

### 8.4 Non-obvious invariants

- **Built-in prefix ordering is cache-load-bearing.** Reordering
  built-in tools between requests busts the server's prompt cache.
  `assembleToolPool` and `mergeAndFilterTools` both re-partition to
  preserve this.
- **Deny rules affect tool visibility, ask/allow rules do not.**
  `filterToolsByDenyRules` only strips tools when the deny rule
  targets the tool itself (no input scope). Input-scoped denies
  stay in the pool and are enforced at call time, because the model
  may need to see the tool to decide not to use it on a denied path.
- **`--dangerously-skip-permissions` still runs the 1g safetyCheck.**
  Bypass mode is not total bypass — bypass-immune safety checks
  still fire. This is how we keep invariants (e.g., no writes
  outside the project root) even when the user has opted out of
  prompting.
- **Auto mode strips dangerous permissions rather than refusing to
  enter.** Entering auto mode silently narrows the allow set;
  exiting restores it. This prevents the user from forgetting that
  they had a blanket `Bash(*)` allow that bypasses the classifier.
- **MCP tool names are the deduplication key.** If a user installs
  two MCP servers that both publish a `read_file` tool, they must
  have distinct server prefixes (`mcp__serverA__read_file` vs
  `mcp__serverB__read_file`). The `mcp__server` wildcard rule
  syntax matches the whole server's tools — useful for blanket
  allow/deny per server.
- **Subagent permission context has `shouldAvoidPermissionPrompts=true`
  for async agents.** An async subagent cannot interactively prompt
  the user; an unmatched tool call implicitly denies rather than
  blocking on a dialog that no one will see.
- **Simple mode (`CLAUDE_CODE_SIMPLE=1`) is hard-coded to three
  tools.** Bash, Read, Edit. This is a documented escape hatch for
  users who want a minimal tool surface (e.g., for deterministic
  benchmarks).
