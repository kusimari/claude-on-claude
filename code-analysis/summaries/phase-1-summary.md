# Phase 1 Summary — Setup, Auth, Tools, MCP, State

Covers reports 01–05: environment setup, trust/auth, tool system, MCP bootstrap, state/config/settings.

---

## 01 — `setup()` and Environment Initialization

**Complexity**: Complex

### Key findings
- `setup()` (src/setup.ts, ~480 LOC) is the interactive-session environment gate; runs after Commander `preAction` → `init()` and before `showSetupScreens()`.
- Twelve phases (A–L): Node-version gate → UDS messaging → teammate snapshot → terminal restore → cwd+hooks snapshot → worktree creation (chdirs mid-function) → background prefetches → ant/attribution hooks → `initSinks()` + `tengu_started` → release-notes prefetch → sandbox safety → prior-session exit replay.
- **Ordering invariants**: `setCwd()` must precede `captureHooksConfigSnapshot()`; `await startUdsMessaging` is intentional so `SessionStart` hooks inherit `CLAUDE_CODE_MESSAGING_SOCKET`; `tengu_started` is the earliest reliable beacon and must not move later.
- `--bare` disables most optional subsystems (UDS, session memory, plugins prefetch, attribution, release notes) but keeps node check, worktree, sandbox safety, sinks.
- Worktree block (Phase F) is ~110 LOC and mutates cwd/originalCwd/projectRoot + clears memory-file caches + re-captures hooks snapshot from the new cwd.

### Critical code locations
- `src/setup.ts:56-66` — signature; `:70-79` node gate; `:89-102` UDS; `:176-285` worktree; `:287-330` background; `:371-381` sinks+beacon; `:396-442` bypass-permissions safety; `:449-476` tengu_exit replay.
- `src/main.tsx:1903-1935` — call site; `setup()` runs in parallel with `getCommands()` + `getAgentDefinitionsWithOverrides()` unless worktree enabled.
- `src/entrypoints/init.ts:57-238` — distinct from setup, runs under Commander preAction for all subcommands.

### Integration points
- **Hooks**: snapshots merged config, registers FileChanged watcher, defers attribution hook registration via `setImmediate`.
- **Worktree** (`utils/worktree.ts`): `createWorktreeForSession` is entry; hook-based or git-backed paths; tmux integration.
- **Analytics**: `initSinks()` drains queued events; idempotent between `init()` and `setup()`.
- **Plugins**: `loadPluginHooks()` + hot-reload subscription.

### Recommendations
- Follow-up deep dives: hooks system (15 files), worktree (~1000 LOC), UDS messaging, plugin loading, teammate/swarm snapshot, attribution defer pattern.

---

## 02 — Trust & Authentication (`showSetupScreens` + auth stack)

**Complexity**: Very Complex

### Key findings
- `showSetupScreens()` (src/interactiveHelpers.tsx:104-298) is the interactive-only trust boundary; CI/print mode bypasses entirely via `main.tsx` short-circuit.
- Fixed dialog order: Onboarding → TrustDialog → MCP-json approvals → CLAUDE.md external-includes → Grove policy → ApproveApiKey → BypassPermissions → AutoMode → DevChannels → ChromeOnboarding.
- Trust persistence: session flag (bootstrap/state) + per-directory disk (`config.projects[path].hasTrustDialogAccepted`); home directory is session-only — never persisted, preventing auto-trust of all cloned repos.
- `computeTrustDialogAccepted()` walks parent directories — any trusted ancestor propagates; memoizes only `true` (so mid-session acceptance is picked up on false re-check).
- Two parallel auth cascades: `getAuthTokenSource()` for OAuth (env → FD → apiKeyHelper → secureStorage) and `getAnthropicApiKeyWithSource()` for API keys; bare mode bypasses most steps.
- OAuth: PKCE flow with localhost callback listener, enterprise `forceLoginOrgUUID` enforced against uncached server profile. 401 handling dedups in-flight refresh by token value; filesystem lock on `~/.claude/` serializes refreshes across processes.

### Critical code locations
- `src/interactiveHelpers.tsx:104-298` — `showSetupScreens()`; `:144-150` — `setSessionTrustAccepted(true)` + `resetGrowthBook()+initializeGrowthBook()`; `:184` — `applyConfigEnvironmentVariables()` post-trust; `:190` — `setImmediate(initializeTelemetryAfterTrust)`.
- `src/components/TrustDialog/TrustDialog.tsx:23+` — dialog; `:31-127` dangerous-feature detection; `:174-177` home-vs-project split; `:257` binary prompt.
- `src/utils/config.ts:697-743` — trust computation/memoization; `:705-743` parent walk.
- `src/utils/auth.ts:100-149` `isAnthropicAuthEnabled`; `:153-206` OAuth cascade; `:226-348` API-key cascade; `:1360-1392` 401 handling; `:1427-1562` refresh with cross-process lock; `:1923-2000` forceLogin validation.
- `src/services/oauth/client.ts:46-144` — PKCE URL build + token exchange; `:146-274` refresh with scope expansion.
- `src/components/ApproveApiKey.tsx` + `auth.ts:1103-1114` — hash-truncated key approval registry.

### Integration points
- **GrowthBook**: auth headers only attached post-trust; `resetGrowthBook()+initializeGrowthBook()` is the handshake.
- **Managed env** (`applyConfigEnvironmentVariables`) and **telemetry** (`otelHeadersHelper`) are trust-gated because both can execute shell commands.
- **GitHub repo mapping** (`updateGithubRepoPathMapping`): trust-gated explicitly to prevent poisoning of teleport mapping.
- **Graceful shutdown**: decline paths call `gracefulShutdownSync(0|1)` — Grove is exit-0, TrustDialog decline is exit-1.

### Recommendations
- Dedicated deep dive on OAuth refresh/401 machinery (`auth.ts:1360-1562`) — cross-process lock semantics, in-flight dedup, scope-expansion refresh, keychain reconciliation path.
- Secure-storage backend layer (`src/utils/secureStorage/*`) — keychain vs plaintext migration.

### Invariants
- Trust precedes env-var application, OAuth helpers, GitHub mapping, GrowthBook init.
- Home directory never persists trust; enterprise `forceLoginOrgUUID` uses uncached profile.

---

## 03 — Tool System Initialization

**Complexity**: Very Complex

### Key findings
- 5-stage tool assembly pipeline: `getAllBaseTools()` → `getTools(context)` → `assembleToolPool(ctx, mcpTools)` → `useMergedTools` (with `initialTools`, coordinator filter) → `getToolForApi()` (JSONSchema serialization, cached).
- **Built-in-prefix ordering is prompt-cache-load-bearing**: `assembleToolPool` and `mergeAndFilterTools` both re-partition `[...builtIn.sort, ...mcp.sort]` — reordering between requests busts server-side prompt cache.
- 8-step permission pipeline (`permissions.ts:1158`): 1a deny → 1b ask → 1c tool.checkPermissions → 1d tool-deny → 1e requiresUserInteraction → 1f content-ask → 1g safetyCheck → 2a bypass → 2b allow → 2c ask-default. Bypass mode still runs safetyCheck (never total bypass).
- Permission modes: `default`, `plan`, `acceptEdits`, `bypassPermissions`, `auto`, `dontAsk`. Auto mode strips dangerous Bash/PowerShell/Task permissions into `strippedDangerousRules` for restore on exit; uses `TRANSCRIPT_CLASSIFIER` with 2s grace race to promote `ask`→`allow`.
- Subagent tool filtering (`filterToolsForAgent`): layered denies — ALL_AGENT_DISALLOWED, CUSTOM_AGENT_DISALLOWED, ASYNC_AGENT_ALLOWED allowlist (excludes MCP), plus `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`.

### Critical code locations
- `src/Tool.ts:380-690` shape; `:757-780` TOOL_DEFAULTS; `:783` `buildTool()`; `:140` empty permission context.
- `src/tools.ts:193` `getAllBaseTools`; `:271` `getTools` (simple mode, deny, REPL, isEnabled filters); `:345` `assembleToolPool`.
- `src/utils/toolPool.ts:55` `mergeAndFilterTools` (pure, no React).
- `src/utils/api.ts:135` `getToolForApi` — strips swarm-only fields, caches zodToJsonSchema output.
- `src/utils/permissions/permissions.ts:473` outer wrapper; `:1158` inner 8-step; `:238` `toolMatchesRule`; `:325` `filterDeniedAgents`.
- `src/utils/permissions/permissionSetup.ts:94-245` dangerous predicates; `:510` strip; `:561` restore; `:597` mode transition; `:872` initialization; `:1078` auto-mode gate verification.
- `src/tools/AgentTool/agentToolUtils.ts:70` `filterToolsForAgent`; `:122` `resolveAgentTools`.

### Integration points
- **MCP**: MCP tools attached via `assembleToolPool`; `mcp__server__tool` namespace; MCP servers ship `inputJSONSchema` verbatim (no Zod conversion).
- **Coordinator mode** (`feature('COORDINATOR_MODE')`): narrows pool to `COORDINATOR_MODE_ALLOWED_TOOLS` + PR activity subscription suffix.
- **Settings**: `permissions.allow/deny/ask` rules, `--allowedTools`/`--disallowedTools`, `additionalDirectories`.
- **Subagents** force `shouldAvoidPermissionPrompts=true` and inherit parent's mode/directories.

### Recommendations
- Deep dives: Bash tool (classifier race, command shape validation), MCP tool generation/router, Agent/Task tool (~2k LOC), Coordinator mode, MemoryTool/TodoWrite, permission rules persistence layer.

### Invariants
- Deny rules strip tools from the visible catalog only if tool-scoped (no input). Input-scoped denies stay in the pool.
- `--dangerously-skip-permissions` still runs 1g safetyCheck.
- Built-in prefix order fixed; partition sort must preserve.
- Simple mode (`CLAUDE_CODE_SIMPLE=1`) = `[Bash, Read, Edit]`.

---

## 04 — MCP Bootstrap

**Complexity**: Very Complex

### Key findings
- MCP = plugin/extensibility vector; 0–100+ servers per session over 7 transports: stdio, sse, http, ws, sse-ide, ws-ide, sdk, claudeai-proxy.
- Config discovery 7 scopes: enterprise (exclusive if present) > user > project (.mcp.json, parent walk) > local > dynamic (--mcp-config/plugin) > claudeai > managed; merged + deduped by signature (stdio JSON-command or URL-unwrapped).
- `connectToServer` (client.ts:595, memoized by `${name}-${JSON.stringify(serverRef)}`) dispatches per transport; on drop (`onclose`) clears cache + re-fetch caches; reconnects with exponential backoff (1s→30s, max 5 attempts) for remote only.
- Tool registration: `mcp__{server}__{tool}` qualified names (except `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`); `inputJSONSchema` passes through verbatim; MCP hints mapped to Tool methods (`readOnlyHint`, `destructiveHint`, `openWorldHint`).
- OAuth auth (`ClaudeAuthProvider`, auth.ts:1376): PKCE + DCR + CIMD (client metadata URL) + XAA (cross-app ID token exchange) + step-up (403 `insufficient_scope` → force PKCE); 15-min `needs-auth` cache TTL prevents probe spam.
- Two-phase load in `useManageMCPConnections`: regular configs concurrent with claude.ai fetch; updates batched every 16ms via `pendingUpdatesRef` to prevent render thrash.

### Critical code locations
- `src/services/mcp/client.ts:595-1647` — `connectToServer`; `:619-865` per-transport setup; `:905-943` in-process (Chrome, Computer Use); `:1221-1402` onerror/onclose; `:1743-1996` `fetchToolsForClient`; `:2226-2403` `getMcpToolsCommandsAndResources`; `:2813/3029` `callMCPTool`.
- `src/services/mcp/auth.ts:1354` `wrapFetchWithStepUpDetection`; `:1376` `ClaudeAuthProvider`; `:1585-1667` XAA + proactive refresh.
- `src/services/mcp/config.ts:1071` `getClaudeCodeMcpConfigs` merge; `:223-266` plugin dedup; `:281-310` claudeai dedup; `:536` policy filter; `:556-616` env expansion; `:202-212` signatures.
- `src/services/mcp/useManageMCPConnections.ts:143` main hook; `:371-464` reconnect backoff; `:618-751` list_changed notifications; `:772-854` stale cleanup; `:858-1024` two-phase load.
- `src/services/mcp/SdkControlTransport.ts:60-109` — JSONRPC bridge CLI↔SDK.
- `src/services/mcp/InProcessTransport.ts` — `createLinkedTransportPair()` for Chrome/Computer Use.
- Main launch wiring: `src/main.tsx:1414-1522` (--mcp-config parsing); `:1803-1814` kickoff; `:2380-2430` post-setup join + prefetch; `:2700-2720` (print mode).

### Integration points
- **Tool system**: MCP tools are regular `Tool` with `isMcp: true`; permission passthrough writes sticky localSettings rule on approval.
- **Plugins**: plugin MCP servers scope=`dynamic` + `pluginSource`, deduped by signature.
- **Secure storage**: OAuth tokens per `serverKey`; `tokens()` called 30-40×/sec → cache hit-rate critical.
- **Message queue**: channel notifications enqueue with `priority:'next'`; `notifications/claude/channel/permission` resolves permission callbacks.
- **Hooks**: `executeElicitationHooks`, `executeElicitationResultHooks`, `executeNotificationHooks`.

### Recommendations
- Deep dives: MCP OAuth/auth flows (2465 LoC), tool call lifecycle (`callMCPTool`→`transformMCPResult`), channel notifications + Kairos, elicitation, plugin-MCP integration.
- Refactor `connectToServer` (>1000 LoC) into per-transport factories.
- Open question: `.mcp.json` file watcher (none detected — requires `/mcp reload`).

### Timeouts (constants)
- `MCP_TIMEOUT` default 30s; `MCP_REQUEST_TIMEOUT_MS=60s`; `MCP_TOOL_TIMEOUT` default ~27.8h; `MCP_AUTH_CACHE_TTL_MS=15min`; `MCP_BATCH_FLUSH_MS=16`; `MAX_RECONNECT_ATTEMPTS=5`.

---

## 05 — State, Config, and Settings

**Complexity**: Complex

### Key findings
- Three distinct layers. **STATE** (`bootstrap/state.ts`): process-scoped module singleton, no persistence, bare getters/setters, bootstrap-isolation eslint-enforced. **Config** (`utils/config.ts`): `~/.claude.json` with write-through cache + file-watcher freshness. **Settings** (`utils/settings/`): 5-source Zod-validated merge — `policy > flag > local > project > user`, read-only at runtime.
- Config save uses file lock (`config.json.lock`), re-reads under lock, auth-loss guard (refuses if `oauthAccount` present in cache but missing in on-disk — GH #3117), timestamped backups (min 60s apart, 5 kept).
- Settings MDM: macOS plist, Windows registry (HKLM > managed-file > HKCU), Linux drop-in dir (`/etc/claude-code/managed-settings.d/*.json` alphabetical); subprocess spawned at `main.tsx` top-level, awaited before first settings access.
- Two env-apply paths: `applySafeConfigEnvironmentVariables()` pre-trust (userSettings + flagSettings + policySettings only, `SAFE_ENV_VARS` allowlist stripping `LD_PRELOAD`/`PATH`), vs `applyConfigEnvironmentVariables()` full-trust.
- Trust computation walks parent dirs; memoizes `true` only so mid-session transitions are detected. Home dir session-only.

### Critical code locations
- `src/bootstrap/state.ts:45-257` type def; `:260-426` `getInitialState`; `:429` STATE init; `:431-1757` getters/setters.
- `src/utils/config.ts:76-136` ProjectConfig; `:183-578` GlobalConfig; `:585-623` defaults; `:797-866` `saveGlobalConfig`; `:912-960` migrations; `:997-1086` load + freshness watcher; `:1153-1329` lock-based write + auth guard + backups; `:1588` `getProjectPathForConfig`; `:697-743` trust computation.
- `src/utils/settings/settings.ts:319-374` `getSettingsForSourceUncached`; `:538` `settingsMergeCustomizer` (array concat+uniq); `:645` `loadSettingsFromDisk`; `:812-868` entry; `:882-910` permission predicates (note `hasAutoModeOptIn` intentionally excludes projectSettings to block RCE).
- `src/utils/settings/mdm/settings.ts:67-78` `startMdmSettingsLoad`; `:124-140` `getMdmSettings` first-source-wins.
- `src/utils/managedEnv.ts:124-178` safe env apply; `:187-199` full env apply.
- `src/main.tsx:653` `enableConfigs()` gate; `:914` `await ensureMdmSettingsLoaded()`; `:2573` full env apply.

### Integration points
- **Bootstrap ordering**: enableConfigs → MDM awaited → first settings access → trust dialog → full env apply → telemetry init (post-trust).
- **Everywhere**: `getSessionId()`, `getMainLoopModelOverride()`, `getProjectRoot()`, `getSettingsWithErrors()` consumed by query.ts, REPL.tsx.
- **Cost tracking**: `addToTotalCostState()`, `modelUsage` restored via `restoreCostStateForSession()` from ProjectConfig.
- **Beta header latches** (`afkModeHeaderLatched`, etc.): sticky-on through session, cleared by `/clear` and `/compact` to protect prompt cache.

### Recommendations
- Follow-up on `/config` UI flow and `applySettingsChange` write path.
- Document the freshness-watcher write-through semantics to avoid double-reads in multi-process tests.

### Invariants
- `projectSettings` never a trusted source for dangerous env vars or auto-mode opt-in.
- Beta header latches must not flip mid-session to preserve prompt cache.
- `migrationVersion` gate prevents N× saveGlobalConfig on startup.

---

## Cross-cutting themes

- **Trust boundary pervades everything**: env vars, GrowthBook auth, telemetry, GitHub mapping, all gated post-trust.
- **Prompt cache discipline**: built-in tool-prefix ordering, beta header latches, `promptCache1hAllowlist` latch — all preserve server-side cache keys.
- **Lazy imports + feature gates**: tools, MCP, plugins use `if (feature('X'))` + dynamic `require()` patterns to enable Bun dead-code elimination.
- **Async gated on sync state**: MDM subprocess, UDS socket, claude.ai MCP fetch all kicked off at top-level then awaited at dependent checkpoints.
- **Config persistence defensive**: filesystem locks, write-through cache + freshness, auth-loss guard, timestamped backups, per-process dedup keys (e.g., OAuth refresh).
- **Subagents = scoped permission context**: force `shouldAvoidPermissionPrompts=true`, `awaitAutomatedChecksBeforeDialog=true`; inherit parent mode + cwd; async agents exclude MCP tools.
