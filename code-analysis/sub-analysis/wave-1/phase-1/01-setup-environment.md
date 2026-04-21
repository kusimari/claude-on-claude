# 01 — `setup()` and Environment Initialization

**Scope**: The `setup()` function in `src/setup.ts` (≈480 LOC) is the
"configure the environment before the REPL renders" choke point for the
default command. It runs after Commander's `preAction` hook (which calls
`init()`) and before `showSetupScreens()` / `replLauncher`. This report
walks the control flow, identifies every side effect it triggers, and
flags subsystems that deserve their own dedicated deep dives.

---

## 1. Call-site wiring (`main.tsx`)

`setup()` is invoked from the top-level `action()` handler after Commander
parses CLI args. The sequence — starting at `main.tsx:1903` — is:

1. `profileCheckpoint('action_before_setup')` — startup profiler marker.
2. `const { setup } = await import('./setup.js')` — lazy import so the
   setup module (and its heavy transitive deps like `worktree.ts`,
   `udsMessaging`, session memory) only loads for commands that need it.
3. `setup()` is kicked off as a non-awaited promise (`setupPromise`) in
   parallel with `getCommands(preSetupCwd)` and
   `getAgentDefinitionsWithOverrides(preSetupCwd)`
   (`main.tsx:1927-1929`). The parallelization comment explicitly notes
   that `setup()`'s ~28 ms is dominated by `startUdsMessaging` (~20 ms
   socket bind), not disk I/O, so it doesn't contend with
   `getCommands()` file reads.
4. Critically, the parallel fan-out is **gated on `!worktreeEnabled`**,
   because `setup()` performs `process.chdir()` when creating a worktree
   and commands/agents need the post-chdir cwd.
5. `await setupPromise` (`main.tsx:1934`) joins back before the action
   handler touches model resolution, trust dialogs, or sinks from the
   REPL side.

`setup()` takes nine parameters: `cwd`, `permissionMode`,
`allowDangerouslySkipPermissions`, `worktreeEnabled`, `worktreeName`,
`tmuxEnabled`, `customSessionId`, `worktreePRNumber`, and
`messagingSocketPath`. Every parameter is a dial from the CLI flags
parsed upstream.

`init()` (in `src/entrypoints/init.ts`) runs earlier via Commander's
`preAction` (`main.tsx:907-967`) and is a distinct function from
`setup()`. That function handles process-wide concerns (config validation,
mTLS/proxy agents, graceful shutdown, telemetry plumbing, upstream proxy
for CCR) that must run for *every* subcommand — `setup()` only runs for
the default "start a session" command. The separation matters: sinks,
for example, are attached in both places (`init.ts` → preAction for
subcommands at `main.tsx:931-935`, and `setup.ts:371` for the default
command) because each code path has a different set of gated features.

---

## 2. Phased control flow inside `setup()`

### Phase A — Node version + session ID (`setup.ts:67-84`)

- `process.version` regex parse, error + `process.exit(1)` if `<18`.
  This is a hard gate that runs before anything else — notably before
  any analytics, because `initSinks()` hasn't attached yet.
- `customSessionId` (from `--session-id <uuid>`) is committed via
  `switchSession(asSessionId(customSessionId))`, mutating bootstrap
  state.

### Phase B — UDS messaging server (`setup.ts:89-102`)

- Gated on `!isBareMode() || messagingSocketPath !== undefined` and
  `feature('UDS_INBOX')`.
- `--bare` deliberately skips the UDS socket because scripted callers
  don't receive injected messages.
- Explicit `--messaging-socket-path` is the escape hatch.
- Dynamic import: `./utils/udsMessaging.js` → `startUdsMessaging(path,
  { isExplicit })`. **This is awaited** because downstream
  `SessionStart` hooks may spawn subprocesses that snapshot
  `process.env.CLAUDE_CODE_MESSAGING_SOCKET` — if it's not exported,
  hooks can't contact the UDS.

### Phase C — Teammate-mode snapshot (`setup.ts:105-110`)

- Gated on `!isBareMode() && isAgentSwarmsEnabled()` (no `--bare`
  escape hatch).
- Dynamic import of `./utils/swarm/backends/teammateModeSnapshot.js`
  → `captureTeammateModeSnapshot()`.
- Freezes the swarm teammate configuration so later hooks see a
  consistent picture.

### Phase D — Terminal backup restoration (`setup.ts:115-158`)

- Interactive sessions only (`!getIsNonInteractiveSession()`).
- iTerm2 backup check (`checkAndRestoreITerm2Backup()`) is gated on
  `isAgentSwarmsEnabled()`.
- Terminal.app backup check is unconditional within the interactive
  branch, wrapped in try/catch so a failed restore doesn't crash setup.
- Both restorations recover from a *prior crashed session* that left
  `defaults` in an inconsistent state.

### Phase E — CWD + hooks snapshot + FileChanged watcher (`setup.ts:160-172`)

- `setCwd(cwd)` — this is "THE" cwd for the session; everything below
  depends on it (file lookups, hook resolution, settings merges).
- `captureHooksConfigSnapshot()` — called via
  `src/utils/hooks/hooksConfigSnapshot.ts:95`. Freezes the merged
  hooks configuration into `initialHooksConfig` so later edits to
  `.claude/settings.json` can't inject hooks mid-session. Respects
  policy knobs (`allowManagedHooksOnly`, `disableAllHooks`).
- `initializeFileChangedWatcher(cwd)` — starts a chokidar watcher
  (see `src/utils/hooks/fileChangedWatcher.ts:28`) that fires
  `FileChanged` / `CwdChanged` hooks when tracked files change. Only
  spins up if the hook config declared any FileChanged matchers.

### Phase F — Worktree creation (`setup.ts:176-285`) ⚠️ long section

If `--worktree` was passed, this is the most involved block in the
function. The flow:

1. Verify capability: `hasWorktreeCreateHook()` OR `await getIsGit()`.
   If neither, `process.exit(1)` with a red error.
2. Resolve slug: `pr-${worktreePRNumber}` > `worktreeName` >
   `getPlanSlug()`.
3. Git path (`inGit === true`):
   - `findCanonicalGitRoot(getCwd())` — resolves nested worktree
     invocations up to the main repo root.
   - If currently inside a sub-worktree, `process.chdir(mainRepoRoot)`
     and `setCwd(mainRepoRoot)` to run git commands from the main repo.
   - Generate `tmuxSessionName` via
     `generateTmuxSessionName(mainRepoRoot, worktreeBranchName(slug))`
     if `--tmux` was passed.
4. Non-git hook path: skip the canonical-root resolution (no git root
   exists), name the tmux session from `getCwd()`.
5. `createWorktreeForSession(sessionId, slug, tmuxSessionName, ...)`
   (implementation in `src/utils/worktree.ts:702`) handles both hook-
   and git-backed paths. Hook-based calls `executeWorktreeCreateHook`
   and trusts whatever path the hook returns.
6. If git succeeded and `tmuxEnabled`, `createTmuxSessionForWorktree`
   is called; a green success or yellow warning is printed.
7. Post-creation state update:
   - `process.chdir(worktreeSession.worktreePath)` +
     `setCwd(worktreeSession.worktreePath)`.
   - `setOriginalCwd(getCwd())` — `originalCwd` is what's shown in
     "cd out" messages.
   - `setProjectRoot(getCwd())` — the worktree becomes the session's
     project (skills, hooks, cron all resolve here).
   - `saveWorktreeState(worktreeSession)` persists for restore.
   - `clearMemoryFileCaches()` — CLAUDE.md cache is tied to
     `originalCwd`; must invalidate.
   - `updateHooksConfigSnapshot()` — re-reads `.claude/settings.json`
     from the worktree and re-captures the hooks snapshot. Comment
     at `setup.ts:280-284` explains that the cache was warmed twice
     from the original dir already, so a fresh read is required.

Worktree is complex enough to warrant a dedicated follow-up analysis
(see §6).

### Phase G — Background/foreground pre-fetches (`setup.ts:287-330`)

Not gated by bare-mode except where noted:

- `initSessionMemory()` — sync; registers a hook. Gate check is lazy.
- `initContextCollapse()` — feature-gated (`CONTEXT_COLLAPSE`),
  dynamic require to hit the linter-enforced lazy-require pattern.
- `lockCurrentVersion()` (voided) — native-installer lock to prevent
  concurrent updates from deleting the running binary.
- `getCommands(getProjectRoot())` (voided) — **pre-warms the command
  cache** so the Promise.all join in `main.tsx:2029` becomes a cache
  hit. Skipped entirely under `skipPluginPrefetch` (bare mode or
  sync plugin install).
- `loadPluginHooks.js` dynamic import → `loadPluginHooks()` +
  `setupPluginHookHotReload()`.

### Phase H — Ant-only + commit attribution + analytics (`setup.ts:336-370`)

- `USER_TYPE === 'ant'` primes `commitAttribution.isInternalModelRepo()`
  — if internal, clears the system-prompt sections cache so
  undercover-mode prompts flip back to OFF.
- `feature('COMMIT_ATTRIBUTION')` → `setImmediate(() => registerAttributionHooks())`.
  Deferred to next tick so the git subprocess spawn happens *after*
  first render, not during setup's microtask window.
- `registerSessionFileAccessHooks()` — analytics for file access during
  a session.
- `feature('TEAMMEM')` → `startTeamMemoryWatcher()`.

### Phase I — Sinks + tengu_started beacon (`setup.ts:371-381`)

- `initSinks()` attaches analytics + error log sinks and drains
  events queued before attach. Idempotent.
- `logEvent('tengu_started', {})` — comment at `setup.ts:372-377`
  explains this is emitted **before** `checkForReleaseNotes` precisely
  because inc-3694 crashed there and swallowed every subsequent event.
- `prefetchApiKeyFromApiKeyHelperIfSafe(getIsNonInteractiveSession())`
  — voided; the "safe" part means "only if trust is already confirmed."

### Phase J — Logo v2 release-notes prefetch (`setup.ts:386-393`)

- Skipped under `--bare`.
- `checkForReleaseNotes(getGlobalConfig().lastReleaseNotesSeen)` is
  awaited; if positive, `getRecentActivity()` pre-reads up to 10
  JSONL session files.

### Phase K — `--dangerously-skip-permissions` sandbox check (`setup.ts:396-442`)

- Root/sudo check (Unix): `process.getuid() === 0` fails with error.
  Sandbox env (`IS_SANDBOX=1` or `CLAUDE_CODE_BUBBLEWRAP`) exempt.
- For `USER_TYPE === 'ant'`, requires Docker/Bubblewrap/sandbox AND
  no internet access. Exempted for `local-agent` and `claude-desktop`
  entrypoints (trusted launchers that pre-approve).

### Phase L — tengu_exit replay (`setup.ts:449-476`)

- Re-emits the *previous* session's exit event using `projectConfig`
  fields that were never cleared on exit. This is how cost/usage
  totals survive across sessions. Used for release health.

Short-circuit: if `NODE_ENV === 'test'`, returns early at
`setup.ts:444-446` before the exit replay.

---

## 3. Key data structures and state touched

| State | Setter | Location |
|---|---|---|
| Session ID | `switchSession(asSessionId(...))` | bootstrap/state.ts |
| cwd | `setCwd(cwd)` | utils/Shell.ts + utils/cwd.ts |
| originalCwd | `setOriginalCwd(getCwd())` | bootstrap/state.ts |
| projectRoot | `setProjectRoot(getCwd())` | bootstrap/state.ts |
| Hooks snapshot | `initialHooksConfig` (module-scoped) | utils/hooks/hooksConfigSnapshot.ts:8 |
| FileChanged watcher | `watcher` (module-scoped `FSWatcher`) | utils/hooks/fileChangedWatcher.ts:14 |
| Worktree session | `currentWorktreeSession` | utils/worktree.ts:156 |
| Analytics sink | `attachAnalyticsSink(...)` | services/analytics/sink.ts |
| Error log sink | via `initializeErrorLogSink()` | utils/errorLogSink.ts |
| Version lock | Filesystem lock file | utils/nativeInstaller/index.ts |

`WorktreeSession` shape (`worktree.ts:140-154`): `originalCwd`,
`worktreePath`, `worktreeName`, optional `worktreeBranch`,
`originalBranch`, `originalHeadCommit`, `sessionId`, optional
`tmuxSessionName`, `hookBased`, `creationDurationMs`,
`usedSparsePaths`.

`HooksSettings` snapshot lives in `utils/settings/types.ts` — the
snapshot is what all downstream hook-execution paths read from via
`getHooksConfigFromSnapshot()`.

---

## 4. Control-flow diagram

```
Commander preAction → init()   [sets up mTLS, proxies, graceful shutdown, telemetry]
        ↓
action() handler                 [parses CLI flags, plugins/skills init]
        ↓
setup() called as promise + parallel getCommands/getAgents
        ↓
[A] Node version gate  ─ exit(1) on <18
[B] UDS messaging      ─ await startUdsMessaging (gated)
[C] Teammate snapshot  ─ (gated: swarms + !bare)
[D] Terminal restore   ─ interactive only
[E] setCwd + hook snapshot + FileChanged watcher
[F] Worktree creation  ─ exit(1) on misuse; chdir on success
[G] Background jobs    ─ session memory, version lock, commands prefetch
[H] Ant + attribution  ─ ant-only feature hooks
[I] initSinks + tengu_started beacon
[J] Logo v2 prefetch   ─ release notes + recent activity (not bare)
[K] Sandbox safety     ─ exit(1) on unsafe bypassPermissions
[L] tengu_exit replay  ─ last session's cost/duration metrics
        ↓
await setupPromise joins   [main.tsx:1934]
        ↓
action() continues: model resolution → trust dialog → replLauncher
```

---

## 5. Integration points with other subsystems

- **`init()` (`entrypoints/init.ts`)**: Sets up mTLS, HTTP proxy
  agents, graceful shutdown, CCR upstream proxy, OAuth hydration,
  telemetry meter, scratchpad dir. `setup()` depends on `init()`
  having run first (Commander `preAction` guarantees this).
- **Hooks system (`utils/hooks/*.ts`)**: `setup()` captures the
  snapshot (E) and registers the FileChanged watcher (E). Attribution
  hooks (H) are registered in `setImmediate`. `setup()` itself does
  **not** execute any hooks — that's the caller's job (main.tsx calls
  `processSessionStartHooks` later).
- **Commands/agents (`commands.ts` / `getAgentDefinitionsWithOverrides`)**:
  `setup()` fires `getCommands()` voided to prime the cache; the
  caller runs it again under `Promise.all` to join. Skipped in bare
  mode.
- **Worktree (`utils/worktree.ts`)**: Mutates cwd, originalCwd,
  projectRoot, clears `clearMemoryFileCaches()`, re-captures hooks.
  Bridges to tmux via `createTmuxSessionForWorktree`.
- **Analytics (`services/analytics/*`)**: `initSinks()` is the first
  point analytics actually flow. Any `logEvent()` before that point
  queues; `initSinks()` drains. `tengu_started` is emitted right
  after sinks attach.
- **Session memory (`services/SessionMemory/sessionMemory.ts`)**:
  `initSessionMemory()` registers a hook (synchronous); gate check
  is lazy so bare mode doesn't pay the cost.
- **Plugins (`utils/plugins/loadPluginHooks.ts`)**: Pre-loads plugin
  hooks so `processSessionStartHooks` has them ready before first
  render. Hot-reload is set up on settings change.
- **Permissions (`utils/permissions/*`)**: `--dangerously-skip-permissions`
  safety gate is inside `setup()` (K), not a separate module.
- **Version management (`utils/nativeInstaller`)**: `lockCurrentVersion()`
  registers a filesystem lock; `initSinks()` happens after so this
  is fire-and-forget.

---

## 6. Follow-up deep dives recommended

The following subsystems are touched by `setup()` but are complex
enough to warrant their own analyses beyond what fits in this report:

1. **Hooks system deep-dive** — `utils/hooks/*.ts` is ~15 files; the
   snapshot capture, allowed-sources logic (managed/plugin/policy),
   and async hook registry all interact in non-trivial ways.
   `captureHooksConfigSnapshot` in particular hides a sizable
   decision tree (policy settings, `allowManagedHooksOnly`,
   `disableAllHooks`, plugin-only customization).
2. **Worktree system deep-dive** — `utils/worktree.ts` is ~1000+
   LOC, spans git-worktree mechanics, hook-based VCS, sparse
   checkout, tmux integration, symlink optimizations, and session
   restoration. The Phase F block in `setup()` is just the thin
   entry layer.
3. **UDS messaging** — `utils/udsMessaging.js` socket protocol, the
   `messagingSocketPath` escape hatch, interaction with
   `replayUserMessages` and stream-json output.
4. **Plugin loading and hot-reload** — `utils/plugins/loadPluginHooks.ts`,
   the "skipPluginPrefetch" race conditions mentioned in the
   comment at `setup.ts:309-320`.
5. **Teammate/swarm snapshotting** — `utils/swarm/backends/teammateModeSnapshot.ts`;
   the snapshot-freeze interacts with kairos/assistant mode init
   upstream of `setup()`.
6. **Attribution system** — `setImmediate` deferral and
   `registerAttributionHooks()` are ant-specific; worth a focused
   look at the gate-plus-defer pattern.

---

## 7. Notable invariants and subtleties

- **Ordering of `setCwd()` before `captureHooksConfigSnapshot()`** is
  explicitly documented at `setup.ts:163-166`. Hooks are resolved
  from the cwd's `.claude/settings.json`, so the snapshot is
  meaningless without the correct cwd first.
- **The `await startUdsMessaging` is intentional**, not a perf miss.
  The comment at `setup.ts:93-99` explains that SessionStart hooks
  need `CLAUDE_CODE_MESSAGING_SOCKET` in their environment, so the
  socket must be bound before hook spawning.
- **`initSinks()` is idempotent** and is called from both `setup()`
  and `init()` (via preAction for subcommands). Both call sites
  matter because subcommand handlers (`doctor`, `mcp`, `plugin`)
  never reach `setup()`.
- **`tengu_started` is positioned carefully** (I) — it's the
  earliest reliable "process started" beacon. A previous bug caused
  it to be silently dropped, so the comment at `setup.ts:372-377`
  warns against moving it later.
- **Worktree (F) mutates cwd mid-function**, and everything in
  Phases G-L runs against the worktree cwd, not the original.
- **NODE_ENV === 'test' short-circuit** at `setup.ts:444-446` skips
  the sandbox check AND tengu_exit replay. Tests that depend on
  the sandbox check running need to stub `process.env.NODE_ENV`.
- **`--bare` (bareMode) disables**: UDS, teammate snapshot, session
  memory, commands/plugins prefetch, commit attribution, session
  file access hooks, team memory watcher, release notes prefetch.
  It preserves: node version check, terminal restore, cwd+hook
  snapshot, worktree creation, version lock, sinks+tengu_started,
  sandbox safety check, tengu_exit replay.

---

## 8. Code references quick index

| Concern | File:line |
|---|---|
| `setup()` signature | `src/setup.ts:56-66` |
| Call site from main | `src/main.tsx:1903-1935` |
| Node version gate | `src/setup.ts:70-79` |
| UDS messaging start | `src/setup.ts:89-102` |
| Hooks snapshot capture | `src/setup.ts:166` → `src/utils/hooks/hooksConfigSnapshot.ts:95` |
| FileChanged watcher init | `src/setup.ts:172` → `src/utils/hooks/fileChangedWatcher.ts:28` |
| Worktree block | `src/setup.ts:176-285` |
| `createWorktreeForSession` | `src/utils/worktree.ts:702-778` |
| Background jobs | `src/setup.ts:288-302` |
| Plugin hooks prefetch | `src/setup.ts:315-329` |
| Attribution hook registration | `src/setup.ts:350-361` |
| `initSinks()` | `src/setup.ts:371` → `src/utils/sinks.ts:13-16` |
| `tengu_started` beacon | `src/setup.ts:378` |
| Bypass-permissions safety | `src/setup.ts:396-442` |
| tengu_exit replay | `src/setup.ts:449-476` |
| `init()` (not `setup()`) | `src/entrypoints/init.ts:57-238` |
