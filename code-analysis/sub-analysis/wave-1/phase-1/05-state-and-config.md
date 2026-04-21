# State, Config, and Settings Architectural Analysis

## Executive Summary

Claude Code's state/config/settings subsystem operates as three distinct but coordinated layers:

- **STATE** (`src/bootstrap/state.ts`): Process-scoped, in-memory variables capturing session identity, telemetry, API request metadata, user interaction timing, and feature flags. Pure module-scope variables, no persistence to disk.
- **config** (`src/utils/config.ts`): Persistent user/project preferences stored in `~/.claude.json`, cached in-memory with fresh-file detection. Per-project overrides under `config.projects[path]`. Atomic writes with lock files.
- **settings** (`src/utils/settings/`): Hierarchical merge of five sources (user/project/local/flag/policy) with validation errors. Read-only at runtime; disk-backed with platform-specific MDM support (macOS plist, Windows registry, Linux JSON files). Merged once per session, cached.

**Precedence**: policy (highest) > flag > local > project > user (lowest). Within settings, policySettings cannot be overridden; flagSettings are immune to project-scoped malice.

**Initialization timing**: enableConfigs() gate required before any config read. Settings load inline with caching during first getSettingsWithErrors(). MDM reads fire async during startup (main.tsx evaluation) and are awaited before first settings access.

---

## 1. State (`src/bootstrap/state.ts`)

### Overview: Plain Module Variables

STATE is a single module-scope object (line 429) initialized at module load, never reimported, never persisted. Getters/setters form the public API.

```ts
const STATE: State = getInitialState()  // Line 429
```

### All Exported Fields: Line-by-line Summary

**Session Identity & Project Context** (lines 45–50, 500–525):
- `getSessionId()` / `switchSession(sessionId, projectDir)` (431, 468): Atomic switch of active session; fires `onSessionSwitch` signal for concurrent session PID tracking. `sessionProjectDir` always changes with `sessionId` to prevent drift (cc:// 34).
- `getProjectRoot()` / `setProjectRoot(cwd)` (511, 523): Set once at startup (including `--worktree` flag). Stable across mid-session worktree entry so skills/history stay anchored.
- `getOriginalCwd()` / `setOriginalCwd()` (500, 515): Initial cwd, normalized for NFC (Unicode). Never updated by EnterWorktreeTool.
- `getCwdState()` / `setCwdState(cwd)` (527, 531): Current working directory; updated during shell operations.

**Cost & Duration Tracking** (lines 51–64, 543–916):
- `totalCostUSD`, `totalAPIDuration`, `totalAPIDurationWithoutRetries`, `totalToolDuration` (51–54): Accumulated via `addToTotalCostState()` (557), `addToTotalDurationState()` (543), `addToToolDuration()` (586). Restored from prior session via `setCostStateForRestore()` (881).
- `modelUsage: { [modelName]: ModelUsage }` (67): Per-model breakdown (input/output/cache tokens, web search requests, cost). Updated by cost tracker.
- `getTurnHookDurationMs()` / `getTurnToolDurationMs()` / `getTurnClassifierDurationMs()` (592, 610, 623): Per-turn metrics; reset between turns (`resetTurnHookDuration()`, 601; `resetTurnToolDuration()`, 614).

**Model & Client Context** (lines 68–82, 838–862):
- `mainLoopModelOverride`, `initialMainLoopModel` (68–69): Model from `--model` flag or user config update. Getters: `getMainLoopModelOverride()` (838), `getInitialMainLoopModel()` (842).
- `modelStrings` (70): Lazy-loaded via `getModelStrings()` (933), `setModelStrings()` (938). Managed by `src/utils/model/modelStrings.ts`.
- `isInteractive` (71): CLI vs non-interactive (IDE, SDK). Set by `setIsInteractive()` (1065).
- `clientType` (80): 'cli', 'claude-vscode', etc. Set by `setClientType()` (1073).

**Telemetry & Observability** (lines 89–109, 948–1055):
- `meter`, `sessionCounter`, `locCounter`, `prCounter`, `commitCounter`, `costCounter`, `tokenCounter`, `codeEditToolDecisionCounter`, `activeTimeCounter` (90–98): OTel counters initialized by `setMeter()` (948). Getters return `null` until set.
- `statsStore` (99): Observer object for perf metrics. Set by `setStatsStore()` (647); read by `getStatsStore()` (641).
- `loggerProvider`, `eventLogger`, `meterProvider`, `tracerProvider` (104–109): OTel infrastructure. Setters at lines 1029, 1037, 1047, 1053.

**API Request Snapshot** (lines 114–120, 1174–1205):
- `lastAPIRequest`, `lastAPIRequestMessages` (114–118): Omit-messages params and exact message array sent to API. Captured for `/share` transcripts. `setLastAPIRequest()` (1174), `getLastAPIRequest()` (1180).
- `lastClassifierRequests` (120): Auto-mode classifier payloads for transcript.

**Session Flags** (lines 122–206):
- `sessionTrustAccepted` (153): Home-directory trust accepted this session (not persisted). Set by `setSessionTrustAccepted()` (1317) from TrustDialog.tsx (175).
- `sessionBypassPermissionsMode` (133): Bypass permissions gate during this session only. `setSessionBypassPermissionsMode()` (1264), `getSessionBypassPermissionsMode()` (1268).
- `sessionPersistenceDisabled` (155): Skip session save to disk. `setSessionPersistenceDisabled()` (1325), `isSessionPersistenceDisabled()` (1329).
- `scheduledTasksEnabled` (137): Cron scheduler active. `setScheduledTasksEnabled()` (1272), `getScheduledTasksEnabled()` (1276).
- `sessionCronTasks` (143): In-memory cron tasks (not in JSON file). `addSessionCronTask()` (1298), `removeSessionCronTasks()` (1307), `getSessionCronTasks()` (1294).
- `sessionCreatedTeams` (149): Teams created this session for cleanup on shutdown. `getSessionCreatedTeams()` (1472).

**Feature Flags & Headers** (lines 77–78, 136–137, 227–242, 1093–1138):
- `strictToolResultPairing` (77): HFI opt-in to fail fast on tool result mismatch instead of synthetic repair.
- `sdkAgentProgressSummariesEnabled` (78): SDK beta. `setSdkAgentProgressSummariesEnabled()` (1081).
- `userMsgOptIn` (79): `sendUserMessage()` tool opt-in. `setUserMsgOptIn()` (1108).
- `kairosActive` (72): Kairos mode detection. `setKairosActive()` (1089).
- `afkModeHeaderLatched`, `fastModeHeaderLatched`, `cacheEditingHeaderLatched`, `thinkingClearLatched` (227–242): Beta header sticky-ons; prevent cache invalidation during session toggles. Getters/setters at lines 1708–1738. Cleared by `clearBetaHeaderLatches()` (1744) on `/clear` or `/compact`.

**Prompt Cache State** (lines 220–225, 1692–1706):
- `promptCache1hAllowlist`, `promptCache1hEligible` (221, 225): Cached GrowthBook 1h TTL allowlist and user eligibility. Latched on first evaluation so mid-session overage doesn't bust server-side prompt cache.

**API Completion Tracking** (lines 248–256, 753–781):
- `lastMainRequestId` (248): Last successful API requestId. `getLastMainRequestId()` (753), `setLastMainRequestId()` (757).
- `lastApiCompletionTimestamp` (252): Date.now() after API success. `getLastApiCompletionTimestamp()` (761), `setLastApiCompletionTimestamp()` (765).
- `pendingPostCompaction` (256): Set after compaction; consumed by API success event. `markPostCompaction()` (771), `consumePostCompaction()` (777).

**Channels & Plugins** (lines 37–40, 213–217, 351, 1239–1262):
- `allowedChannels` (213): Channel servers from `--channels` flag (plugin marketplace or arbitrary servers). `getAllowedChannels()` (1676), `setAllowedChannels()` (1680).
- `hasDevChannels` (217): True if any entry came from `--dangerously-load-development-channels`. `getHasDevChannels()` (1684), `setHasDevChannels()` (1688).
- `inlinePlugins` (127): `--plugin-dir` session-only plugins. `setInlinePlugins()` (1239), `getInlinePlugins()` (1243).
- `chromeFlagOverride` (129): Explicit `--chrome` / `--no-chrome` flag. `setChromeFlagOverride()` (1247), `getChromeFlagOverride()` (1251).
- `useCoworkPlugins` (131): `--cowork` flag or env var to load from cowork_plugins dir. `setUseCoworkPlugins()` (1255); calls `resetSettingsCache()` (1257).

**Additional Context** (lines 165–207):
- `initJsonSchema`, `registeredHooks` (165, 167): SDK init schema and hook callbacks. `setInitJsonSchema()` (1411), `registerHookCallbacks()` (1419), `clearRegisteredHooks()` (1442), `clearRegisteredPluginHooks()` (1446).
- `planSlugCache` (169): Map of sessionId → slug for wordified plan names. `getPlanSlugCache()` (1468).
- `invokedSkills` (178): Preserved across compaction. `addInvokedSkill()` (1510), `clearInvokedSkills()` (1543).
- `slowOperations` (189): Dev-bar display (ant-only). `addSlowOperation()` (1569), `getSlowOperations()` (1595).
- `mainThreadAgentType` (197): `--agent` flag or settings value. `getMainThreadAgentType()` (1623), `setMainThreadAgentType()` (1627).
- `isRemoteMode` (199): `--remote` flag. `getIsRemoteMode()` (1631), `setIsRemoteMode()` (1635).
- `directConnectServerUrl` (201): For header display. `getDirectConnectServerUrl()` (535), `setDirectConnectServerUrl()` (539).

**Memory & Caching** (lines 203, 219–222):
- `lastEmittedDate` (205): Detect midnight date changes. `getLastEmittedDate()` (1658), `setLastEmittedDate()` (1662).
- `systemPromptSectionCache`, `additionalDirectoriesForClaudeMd` (203, 207): System prompt sections (null = not cached, string = cached content). `getSystemPromptSectionCache()` (1641), `setSystemPromptSectionCacheEntry()` (1645), `clearSystemPromptSectionState()` (1652).

**CLI Configuration Flags** (lines 84–88, 1132–1172):
- `flagSettingsPath`, `flagSettingsInline` (84, 85): `--settings file.json` or SDK inline settings. `getFlagSettingsPath()` (1132), `setFlagSettingsPath()` (1136), `getFlagSettingsInline()` (1140), `setFlagSettingsInline()` (1144).
- `allowedSettingSources` (85): `--setting-sources` filter. `getAllowedSettingSources()` (1226), `setAllowedSettingSources()` (1230).
- `sessionIngressToken`, `oauthTokenFromFd`, `apiKeyFromFd` (86–88): Auth tokens from stdin. `setSessionIngressToken()` (1154), `setOauthTokenFromFd()` (1162), `setApiKeyFromFd()` (1170).

**Agent Color & Notifications** (lines 111–112, 1128):
- `agentColorMap`, `agentColorIndex` (111–112): Teammate color assignment. `getAgentColorMap()` (1128).

**Error Logging** (lines 125, 1215–1224):
- `inMemoryErrorLog` (125): Circular buffer (max 100 entries). `addToInMemoryErrorLog()` (1215).

### State Sharing Mechanism

**No React Context or Store**: STATE is a bare module singleton (not a Redux store, Zustand, or Context). Callers import getters/setters directly:

```ts
import { getSessionId, setIsInteractive } from 'src/bootstrap/state.js'
```

**Bootstrap Leaf**: `src/bootstrap/state.ts` is intentionally a DAG leaf (no heavy imports). Circular-dependency rule enforced via eslint custom rule `bootstrap-isolation` (comment at line 17).

**Test Reset**: `resetStateForTests()` (918) reconstructs STATE from defaults in test mode only. Session-only transient caches (scroll drain, interaction time) reset separately since they live outside STATE.

---

## 2. Config (`src/utils/config.ts`)

### Shape: GlobalConfig & ProjectConfig

**GlobalConfig** (lines 183–578):
- **Top-level structure**: In-memory cache + file at `~/.claude.json` (per `getGlobalClaudeFile()`).
- **Scope**: User-wide, shared across all projects. Covers onboarding state, theme, notifications, API key cache, feature gates.
- **Key fields**:
  - `numStartups`: Onboarding trigger (line 189).
  - `hasCompletedOnboarding`, `lastOnboardingVersion`, `lastReleaseNotesSeen` (198–202): Versioned onboarding control (MACRO.VERSION).
  - `theme` (197), `editorMode` (231), `verbose` (219): UI preferences.
  - `projects: Record<string, ProjectConfig>` (188): Per-project overrides keyed by normalized git root or cwd.
  - `oauthAccount` (229): Cached OAuth account info (accountUuid, email, org).
  - `autoUpdates`, `installMethod` (191, 190): Legacy `autoUpdaterStatus` migrated (lines 912–960).
  - `cachedStatsigGates`, `cachedDynamicConfigs`, `cachedGrowthBookFeatures` (441–449): Feature-gate cache.
  - `env: { [key]: string }` (239): DEPRECATED — settings.env is preferred (line 1235).
  - `migrationVersion` (577): Tracks last migration run to avoid 11× saveGlobalConfig calls on startup.

**ProjectConfig** (lines 76–136):
- **Scope**: Per-project entries under `globalConfig.projects[normalizedPath]`. Path normalized via `normalizePathForConfigKey()`.
- **Persistence**: Computed via `getProjectPathForConfig()` (1588): git root if in repo, else `originalCwd`.
- **Key fields**:
  - `allowedTools` (77): List of permitted tool IDs.
  - `hasTrustDialogAccepted` (111): Trust state per directory (inherited via parent walk in `computeTrustDialogAccepted()`, line 705).
  - `hasCompletedProjectOnboarding` (113): Project-level first-run flag.
  - `enabledMcpServers`, `disabledMcpServers` (124, 122): MCP server toggles (migrated to settings but kept for backward compat, line 118).
  - `lastAPIDuration`, `lastCost`, `lastSessionId`, `modelUsage` (80–106): Cost tracking restored via `restoreCostStateForSession()`.
  - `activeWorktreeSession` (126): Current worktree metadata for session resume.
  - `remoteControlSpawnMode` (135): `same-dir` or `worktree` for multi-session `claude remote-control`.

### File I/O: Load & Save

**Location**: `~/.claude.json` (via `getGlobalClaudeFile()` from env module).

**Loading** (`getGlobalConfig()`, line 1044):
- **Cache-first path** (line 1052): After startup, pure memory read always hits. Writes go write-through (line 1039). Other processes' writes picked up by freshness watcher (line 997).
- **Startup load** (line 1060): Sync read + parse on first call. Stat before read to detect concurrent writes mid-file (line 1062). Returns defaults on missing file (ENOENT).
- **Config freshness watcher** (line 997): `fs.watchFile()` polls every 1000ms (CONFIG_FRESHNESS_POLL_MS, 992). Async re-read on mtime change. Skips re-reads when our own write-through advances cache.mtime beyond file mtime (line 1009).
- **Parsing** (line 1068): JSON parse with BOM strip (PowerShell 5.x), merge with defaults. Corrupted config returns defaults + backup restoration guidance (lines 1456–1579). `ConfigParseError` thrown if `throwOnInvalid=true` (line 1466).

**Saving** (`saveGlobalConfig(updater)`, line 797):
- **Updater pattern** (line 798): Caller receives current config, returns merged result. No changes (same reference) → skip write.
- **Lock-based write** (`saveConfigWithLock()`, line 1153):
  - File lock at `config.json.lock` (1169). Contention logged to analytics (1183).
  - Re-read under lock (line 1216): Guard against concurrent writes from other processes mid-file.
  - Auth-loss guard (line 1217): If current config missing `oauthAccount` that cache has, refuse write (prevents wiping auth, GH #3117).
  - Timestamped backup (line 1241): Before write, copy to `~/.claude/backups/config.json.backup.<timestamp>`. Min 60s interval (1263), keep 5 most recent (1284).
  - Write-through cache (line 1039): After successful write, cache.mtime = Date.now() (overshoots file mtime) so freshness watcher skips re-read.
- **Fallback** (line 840): If lock fails, retry without lock. Auth-loss guard still applied to prevent defaults clobbering good cache.
- **Write-count tracking** (line 1321, 881): `globalConfigWriteCount` incremented per write; exposed for ant dev diagnostics (inc-4552) to detect anomalous write rates.

**Migrations** (`migrateConfigFields()`, line 912):
- `autoUpdaterStatus` → `installMethod` + `autoUpdates` (line 921).
- `history` field removed from projects (function `removeProjectHistory()`, 966). Migrated to history.jsonl per project.

### Trust Dialog State

**Trust Computation** (`checkHasTrustDialogAccepted()`, line 697):
- **Session trust** (line 709): Checked first; home-directory trust not persisted to disk.
- **Project path walk** (line 728): Start from cwd, walk up parents. Any ancestor with `hasTrustDialogAccepted=true` → trust granted.
- **Memoization** (line 702): Only caches `true` (latches on). `false` re-checked every call so mid-session trust acceptance is picked up.
- **`isPathTrusted(dir)`** (line 752): Check arbitrary directory without session trust or memoization; used by /assistant tool when user types path.

### Versioning: Macro & Migration

**MACRO.VERSION** (line 36 in interactiveHelpers.tsx):
```ts
lastOnboardingVersion: MACRO.VERSION
```
Recorded when onboarding completes (completeOnboarding(), line 33). Compared to MIN_VERSION_REQUIRING_ONBOARDING_RESET to decide if re-onboarding needed.

**Migration Version** (line 1347):
```ts
if (didWrite) {
  reportConfigCacheStats()
}
```
Tracks `config.migrationVersion`; `runMigrations()` skips all sync migrations once current version written, avoiding 11× save calls per startup.

---

## 3. Settings (`src/utils/settings/`)

### Directory Structure

```
settings/
├── settings.ts                 # Main entry: getSettings(), merge logic
├── types.ts                    # SettingsJson schema (Zod)
├── constants.ts                # Setting source names & display helpers
├── validation.ts               # Error formatting, permission validation
├── allErrors.ts                # Combines settings + MCP errors
├── settingsCache.ts            # Session-level caching
├── changeDetector.ts           # File watcher for mid-session changes
├── internalWrites.ts           # Guard against recursive applies
├── applySettingsChange.ts      # Apply/save edited setting
├── managedPath.ts              # Platform-specific managed file paths
├── permissionValidation.ts     # Permission rule schemas
├── mdm/
│   ├── settings.ts             # MDM/HKCU read, first-source-wins
│   ├── rawRead.ts              # Subprocess I/O (fires at main.tsx eval)
│   ├── constants.ts            # macOS plist, Windows registry paths
```

### Precedence: Five-Layer Merge

```
policySettings (highest, cannot override)
├── flagSettings (--settings CLI / SDK inline)
├── localSettings (~/.claude/local.json or .claude/local.json, project, gitignored)
├── projectSettings (.claude/settings.json, project, tracked)
└── userSettings (~/.claude/settings.json, global)
```

**Merge Logic** (`settingsMergeCustomizer()`, line 538): Custom deep merge for arrays (concat, dedupe with `uniq()`). Hashes prevent re-merge of same keys.

**Enabled Sources** (`getEnabledSettingSources()`, line 159): Constrained by `allowedSettingSources` in STATE (set via `--setting-sources` flag). policySettings and flagSettings always enabled.

**Setting Source Paths** (`getSettingsFilePathForSource()`, line 274):
- `userSettings`: `~/.claude/settings.json` (via `getClaudeConfigHomeDir()`)
- `projectSettings`: `.claude/settings.json` (in project root)
- `localSettings`: `.claude/local.json` (in project root, gitignored)
- `flagSettings`: Loaded from `--settings file.json` or SDK `settings` field (line 84 in config.ts)
- `policySettings`: Managed-settings.json + MDM (via `getManagedFilePath()`) or remote (via syncCache)

### Load Flow: Startup → Runtime

**Startup** (main.tsx evaluation, ~line 90):
1. `fireRawRead()` in `src/bootstrap/cli.tsx` or equivalently `startMdmSettingsLoad()`: Spawn subprocess to read Windows registry / macOS plist (async, no blocking).
2. `enableConfigs()` (main.tsx ~650): Gate at which config reads become allowed. Validates global config.
3. `applySafeConfigEnvironmentVariables()` (managedEnv.ts, line 124): Apply only trusted env vars (userSettings, flagSettings, policySettings) before trust dialog. Project-scoped env vars stripped (ANTHROPIC_BASE_URL, LD_PRELOAD).

**First Settings Access** (during onboarding/first API call):
1. `getSettingsWithErrors()` (line 856): Calls `loadSettingsFromDisk()` (645) if cache empty.
2. `loadSettingsFromDisk()` (645): Read + merge all enabled sources with validation.
3. Result cached in `getSessionSettingsCache()` (session-level, line 858).
4. `startMdmSettingsLoad()` awaited before first `getSettingsWithErrors()` call (main.tsx ~914).

**Cache Invalidation**:
- `resetSettingsCache()` (settingsCache.ts): Clears all caches. Called by:
  - `setUseCoworkPlugins()` (state.ts, 1257)
  - `getSettingsWithSources()` (837) — reads fresh from disk for `/config` UI
  - File change detector when settings file modified mid-session

**Per-Source Cache** (settingsCache.ts):
```ts
getCachedSettingsForSource(source)    // Line ~30
setCachedSettingsForSource(source)
```
Each source cached separately; merge refreshes session cache.

### Helper Functions: Settings Queries

**Permission Predicate Checks** (settings.ts):
- `hasSkipDangerousModePermissionPrompt()` (882): Check if user/local/flag/policy has opt-ed into bypass mode (skips dialog).
- `hasAutoModeOptIn()` (896): Check if `skipAutoPermissionPrompt` set. Intentionally excludes `projectSettings` to prevent RCE via committed .claude/settings.json.

**Mode Helpers** (settings.ts):
- `getUseAutoModeDuringPlan()` (918): Check if auto mode persists during plan.
- `getAutoModeConfig()` (936): Return full auto-mode config object.

**Raw File Check** (settings.ts):
- `rawSettingsContainsKey(key)` (984): Scan all raw settings files (before validation) to detect user intent even if validation failed.

### Schemas: Zod Validation

**Types** (types.ts, line 1+):
- `SettingsJson`: Unvalidated JSON structure (catch-all record).
- `SettingsSchema` (Zod, imported from types.ts): Validates permissions, env vars, hooks, MCP servers, etc.

**Permission Validation** (permissionValidation.ts, line 1+):
- `PermissionRuleSchema`: Validates allow/deny/ask rules (tools, commands, scripts, etc.).
- `filterInvalidPermissionRules()`: Remove rules that fail validation.

**Environment Variables** (types.ts, line 35):
```ts
EnvironmentVariablesSchema = z.record(z.string(), z.coerce.string())
```
Coerces any value to string.

### Managed Settings (Enterprise/MDM)

**File Locations**:
- **macOS**: Plist at `/Library/Managed Preferences/com.anthropic.claudecode.plist` (MDM only, not user editable).
- **Windows**: Registry keys:
  - `HKLM\SOFTWARE\Policies\ClaudeCode` (admin-only, highest priority)
  - `HKCU\SOFTWARE\Policies\ClaudeCode` (user-writable, lowest priority within policy)
- **Linux**: `/etc/claude-code/managed-settings.json` + drop-in dir `/etc/claude-code/managed-settings.d/*.json` (sorted alphabetically).

**First-Source-Wins** (mdm/settings.ts):
1. Remote managed settings (via API, if eligible, line 1+)
2. HKLM (Windows admin registry)
3. managed-settings.json (local file)
4. HKCU (Windows user registry)
5. (none) → fallback to non-policy sources

**Drop-In Pattern** (settings.ts, line 91): managed-settings.d/*.json files merged in alphabetical order (systemd convention). Base file lowest precedence, drop-ins override.

**Async Load** (mdm/rawRead.ts): Subprocess spawned at main.tsx evaluation. Read promise awaited before first settings access (main.tsx ~914, `ensureMdmSettingsLoaded()`).

---

## 4. Environment Variables: Trusted vs Untrusted Contexts

**`applyConfigEnvironmentVariables()` Flow** (managedEnv.ts, line 187):
- Apply global config env (user-wide).
- Apply merged settings env (all sources).
- Clear caches: CA certs, mTLS, proxy. Rebuild agents.

**`applySafeConfigEnvironmentVariables()` Flow** (line 124):
- **Pre-trust phase**: Apply trusted sources only (userSettings, flagSettings, policySettings). NO project-scoped env vars.
- **Eligibility check** (line 157): `isRemoteManagedSettingsEligible()` after userSettings applied (reads CLAUDE_CODE_USE_BEDROCK, ANTHROPIC_BASE_URL).
- **Policy env** (line 161): Apply policySettings env (highest priority).
- **Safe-only pass** (line 172): From merged settings, apply only SAFE_ENV_VARS allowlist (e.g., NODE_EXTRA_CA_CERTS, not LD_PRELOAD).

**SAFE_ENV_VARS Allowlist** (managedEnvConstants.ts): Curated list of non-dangerous vars from project settings. Prevents `LD_PRELOAD` / `PATH` injection from malicious .claude/settings.json.

**SSH Tunnel Exception** (line 23): ANTHROPIC_UNIX_SOCKET routes auth through forwarded socket; don't let project settings clobber it.

**Host-Managed Provider** (line 45): If `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` set, strip provider-routing vars from settings so user can't redirect to attacker backend.

**CCD Spawn Env Keys** (line 69): Capture keys present at process spawn (Desktop host orchestration). Settings env must not override them (OTEL_LOGS_EXPORTER would corrupt stdio transport).

---

## 5. Initialization Ordering

### main.tsx (~line 1–250)

1. **Line 168**: Import STATE getters/setters.
2. **Line 56**: Import `applyConfigEnvironmentVariables`.
3. **cli.tsx evaluation** (~top-level): If not fork, `fireRawRead()` spawns MDM subprocess async.
4. **Line 812**: `setIsInteractive(isInteractive)` based on TTY detection.
5. **Line 834**: `setClientType(type)` from env / config.
6. **Line 1695**: `setAllowedChannels(channelEntries)` from `--channels` flag.

### main.tsx (~line 650)

7. **Line 653**: `enableConfigs()` gate — config reads now allowed. Validates global config.

### main.tsx (~line 914)

8. **Line 914**: `await ensureMdmSettingsLoaded()` — wait for MDM subprocess.

### main.tsx (~line 2114)

9. **Line 2114**: `setInitialMainLoopModel()` from `getUserSpecifiedModelSetting() || null`.

### showSetupScreens() (interactiveHelpers.tsx, ~line 130+)

10. **Line 144**: Trust dialog → `setSessionTrustAccepted(true)` if accepted.
11. **Line 313**: `setStatsStore(stats)` after telemetry initialized.
12. **Lines 267, 279**: `setAllowedChannels()`, `setHasDevChannels()` for dev channels if confirmed.

### Telemetry Init (entrypoints/init.ts, ~line 1+)

13. `initializeTelemetryAfterTrust()`: Set meter, loggers, tracers.
    - Called after trust dialog from showSetupScreens() → `initializeTelemetryAfterTrust()` (line 10).

### Post-Render (main.tsx, ~line 2573+)

14. **Line 2573**: `applyConfigEnvironmentVariables()` after Ink render setup (full env applied, trust established).

---

## 6. STATE vs Config vs Settings: Conceptual Split

| Aspect | STATE | Config | Settings |
|--------|-------|--------|----------|
| **Persistence** | None (process-scoped) | `~/.claude.json` | `~/.claude/settings.json` + platform MDM |
| **Scope** | Session-only | User-wide, + per-project overrides | Hierarchical merge (5 sources) |
| **Mutability** | Mutable via setters | Read-mostly; saved via `saveGlobalConfig()` | Read-only at runtime (no setter API) |
| **Cache** | Memory (singleton) | Write-through + freshness watcher | Session-level (per-source + merged) |
| **Access** | Getters/setters | `getGlobalConfig()` / `saveGlobalConfig()` | `getSettingsWithErrors()` / `getSettingsForSource()` |
| **Use Case** | Session telemetry, model choice, trust flags | Onboarding state, theme, cost tracking, auth | Permissions, env vars, hooks, MCP |
| **Validation** | Implicit (TS types) | Structural (not validated) | Zod schema validation + error aggregation |

**When to use which**:
- **Need runtime state during session?** → STATE (e.g., current session ID, model override, telemetry).
- **Need user/project preference?** → config (e.g., theme, completed onboarding, trust status).
- **Need operational rules?** → settings (e.g., which tools allowed, hooks, env vars, permission mode).

---

## 7. Critical Fields for Agent Loop

Grep results from query.ts and REPL.tsx (estimated—see earlier bash output):
- `getSessionId()` (431): Identify current session for history/skill scope.
- `getMainLoopModelOverride()` / `getInitialMainLoopModel()` (838, 842): API model selection.
- `getTotalCostUSD()` (566): Display running cost to user.
- `getTotalAPIDuration()` (570): Accumulate API time.
- `getTotalToolDuration()` (582): Tool execution time.
- `getTurnToolCount()` / `getTurnHookCount()` / `getTurnClassifierCount()` (619, 606, 637): Per-turn metrics.
- `getIsInteractive()` (1061): Decide if prompts/spinners should render.
- `getModelUsage()` / `getUsageForModel()` (826, 830): Per-model token breakdown.
- `getSettings_DEPRECATED()` / `getSettingsWithErrors()` (820, 856): Permission checks, mode (auto/normal).
- `getProjectRoot()` (511): Skill/history anchoring.

---

## Files & Line References (Summary)

| Component | File | Key Lines |
|-----------|------|-----------|
| State fields | `src/bootstrap/state.ts` | 45–257 (type def), 429 (init), 431–1757 (getters/setters) |
| State init | `src/bootstrap/state.ts` | 260–426 (`getInitialState()`) |
| GlobalConfig shape | `src/utils/config.ts` | 183–578 (type def), 585–623 (defaults) |
| ProjectConfig shape | `src/utils/config.ts` | 76–136 (type def), 138–148 (defaults) |
| Config load | `src/utils/config.ts` | 1044–1086 (`getGlobalConfig()`) |
| Config save | `src/utils/config.ts` | 797–866 (`saveGlobalConfig()`) |
| Config lock | `src/utils/config.ts` | 1153–1329 (`saveConfigWithLock()`) |
| Trust check | `src/utils/config.ts` | 697–743 (`checkHasTrustDialogAccepted()`, `computeTrustDialogAccepted()`) |
| Settings loader | `src/utils/settings/settings.ts` | 812–868 (`getInitialSettings()`, `getSettingsWithErrors()`) |
| Settings merge | `src/utils/settings/settings.ts` | 319–374 (`getSettingsForSourceUncached()`) |
| Settings sources | `src/utils/settings/constants.ts` | 7–22 (precedence), 159–167 (`getEnabledSettingSources()`) |
| Perms check | `src/utils/settings/settings.ts` | 882–910 (`hasSkipDangerousModePermissionPrompt()`, `hasAutoModeOptIn()`) |
| MDM load | `src/utils/settings/mdm/settings.ts` | 67–78 (`startMdmSettingsLoad()`), 124–140 (`getMdmSettings()`) |
| Safe env apply | `src/utils/managedEnv.ts` | 124–178 (`applySafeConfigEnvironmentVariables()`) |
| Full env apply | `src/utils/managedEnv.ts` | 187–199 (`applyConfigEnvironmentVariables()`) |
| Setup screens | `src/interactiveHelpers.tsx` | ~line 130+ (init calls) |
| Main init | `src/main.tsx` | 812, 834, 1695, 653, 914, 2114, 2573 (state setters + config/settings access) |

