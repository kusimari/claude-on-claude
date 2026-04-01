# Claude Code CLI Architecture

Concise analysis of the codebase structure and launch patterns.

## 1. Overview

TypeScript CLI built with Bun. Single package under `src/`. Three main modes:
- **Fast-path commands**: Specialized utilities that load minimal modules and exit
- **REPL mode**: Full interactive terminal UI with tools, query engine, and background processes
- **Inline commands**: Trivial operations like `--version`

### High-Level Flow

```
LAUNCH: cli.tsx fast-path router → bootstrap → main.tsx → setup() → REPL
INTERACTIVE: User prompt → query() async generator → API + tools → loop until done
BACKGROUND: MCP connections, analytics, session persistence, plugin hot-reload
```

### Key Files

| Component | Files | Purpose |
|-----------|-------|---------|
| **Entry** | `entrypoints/cli.tsx` | Fast-path router, bootstrap |
| **Main** | `main.tsx` (804KB) | Commander.js setup, pre-REPL initialization |
| **Bootstrap** | `bootstrap/state.ts` (56KB) | Global STATE singleton (~80 fields) |
| **Config** | `utils/config.ts` | `~/.claude/config.json` reader with guard |
| **Settings** | `utils/settings/` | Multi-source settings merge |
| **REPL** | `screens/REPL.tsx` | Interactive prompt → query → render loop |
| **Query** | `query.ts` (69KB) | API call → stream → tool execution loop |
| **Tools** | `Tool.ts`, `tools/` | ~40 tool implementations |
| **UI** | `ink/`, `components/` | Custom terminal renderer |

---

## 2. Launch Pipeline: Fast-Path Router

**Pattern**: `claude <command>` → parse argv → route to handler → execute

### Command Classification

```
claude <command> [args...]
│
├─ TRIVIAL (inline logic)
│  ├─ --version → print MACRO.VERSION, exit
│  ├─ --update/--upgrade → argv redirect
│  └─ --bare → set env var
│
├─ FAST-PATHS (specialized utilities)
│  ├─ daemon → daemon/main.js
│  ├─ --daemon-worker → daemon/workerRegistry.js
│  ├─ bridge/remote-control → bridge/bridgeMain.js
│  ├─ ps/logs/attach/kill → cli/bg.js
│  ├─ new/list/reply → cli/handlers/templateJobs.js
│  ├─ environment-runner → environment-runner/main.js
│  ├─ --worktree --tmux → utils/worktree.js
│  └─ [8+ more MCP/utility commands]
│
└─ NORMAL PATH (no match) → main.tsx → REPL
```

### Fast-Path Pattern

```typescript
if (feature('FLAG') && args[0] === 'command') {
  profileCheckpoint('cli_command_path')
  const { enableConfigs } = await import('../utils/config.js')
  enableConfigs()  // Optional - unlocks ~/.claude/config.json
  const { commandMain } = await import('../command/main.js')
  await commandMain(args.slice(1))
  return  // Exit - don't continue to REPL
}
```

**Why "fast-path"?** Minimal imports, early exit, no REPL overhead. Only the normal path loads 804KB main.tsx.

### Complete Fast-Path Map

| Command(s) | Config? | Target File | Notes |
|------------|:-------:|-------------|--------|
| `--version` | ❌ | Inline | Zero imports - just print & exit |
| `--daemon-worker` | ❌ | `daemon/workerRegistry.js` | Leanest path |
| `daemon` | ✅ | `daemon/main.js` | Needs config for analytics |
| `bridge`/`remote-control` | ✅ | `bridge/bridgeMain.js` | Complex: auth + policy checks |
| `ps`/`logs`/`attach`/`kill` | ✅ | `cli/bg.js` | Session registry access |
| `new`/`list`/`reply` | ❌ | `cli/handlers/templateJobs.js` | Standalone templates |
| `--dump-system-prompt` | ✅ | `constants/prompts.js` | Model preferences |
| `--worktree --tmux` | ✅ | `utils/worktree.js` | Can fall through to normal CLI |
| MCP servers | ❌ | Various `mcpServer.js` | Headless utilities |

---

## 3. Bootstrap & Configuration

### Bootstrap State Creation

Every fast-path (except `--version`) loads `startupProfiler.js` → triggers `bootstrap/state.ts`:

```typescript
// Module evaluation side effect:
const STATE: State = getInitialState()

function getInitialState(): State {
  const rawCwd = cwd()
  const resolvedCwd = realpathSync(rawCwd).normalize('NFC')
  return {
    originalCwd: resolvedCwd,
    projectRoot: resolvedCwd,
    cwd: resolvedCwd,
    sessionId: randomUUID(),    // Fresh UUID per session
    startTime: Date.now(),      // Process start timestamp
    /* ...75+ other fields */
  }
}
```

**STATE contains**: identity (cwd, sessionId), costs/timing, model overrides, telemetry objects, session flags, cache latches.

### Config vs Settings - Two Different Systems

**Config (`~/.claude/config.json`)** - User/project data:
- **Auth**: `oauthAccount`, `primaryApiKey`, `customApiKeyResponses`
- **Projects**: Per-directory `allowedTools`, `mcpServers`, `hasTrustDialogAccepted`
- **User prefs**: `theme`, `verbose`, `onboardingComplete`, `autoUpdates`
- **MCP**: Global `mcpServers`, connection history

**Settings (`.claude/settings.json` files)** - Operational control:
- **Permissions**: Tool allow/deny rules, permission modes
- **Environment**: Environment variables, auth helpers
- **Enterprise**: Model allowlists, policy restrictions
- **Hooks**: Custom shell commands on events
- **MCP**: Server configurations (separate from config)

### Config System - Eager Loading

`enableConfigs()` unlocks immediate access to `~/.claude/config.json`:

```typescript
let configReadingAllowed = false

export function enableConfigs(): void {
  configReadingAllowed = true  // Remove guard
  getConfig(getGlobalClaudeFile(), createDefaultGlobalConfig, true)  // Load now
}
```

**Fast-paths that need config**:
- `daemon`: Analytics + session management
- `bridge`: OAuth tokens for authentication
- `ps/logs/attach/kill`: Session registry access
- `--worktree --tmux`: Worktree preferences

### Settings System - Lazy Loading

Settings loaded on first access via `getInitialSettings()`:

```typescript
// Multi-source merge (low→high priority):
plugins < userSettings < projectSettings < localSettings < flagSettings < policySettings
           ~/.claude/    .claude/           .claude/          --settings     managed/remote
           settings.json settings.json      settings.local.json
```

**Fast-paths that use settings**:
- `update`: `getInitialSettings()?.autoUpdatesChannel` (update channel preference)
- `auth`: `settings.forceLoginMethod` (enterprise login policy)

**Fast-paths that skip settings**:
- MCP servers: Standalone utilities, no policies needed
- `--daemon-worker`: Supervisor handles all policy
- Environment runners: Headless, no user preferences

### When Each Loads

```
Fast-path timing:
├─ Profiler loads → STATE created (always except --version)
├─ enableConfigs() → config.json loaded (if path calls it)
└─ getInitialSettings() → settings merged (lazy, on first access)

REPL timing:
├─ All above happens during setup()
├─ Settings accessed throughout: app state, agent selection, permissions
└─ Both config and settings needed for full interactive session
```

---

## 4. Execution Flows

### Fast-Path Flow
```
cli.tsx:main()
├─ args = process.argv.slice(2)
├─ if (--version) → console.log + return
├─ import startupProfiler.js → STATE created
├─ feature('FLAG') && command match
├─ enableConfigs() (maybe)
├─ import command/main.js
├─ await commandMain(args)
└─ return (exit)
```

### Normal Path Flow (REPL)
```
cli.tsx:main()
├─ startCapturingEarlyInput()
├─ import('../main.js') → 804KB + ~150 imports
├─ main() → Commander.js program
├─ .action() → extract options, build tool context
├─ setup() → cwd, hooks, worktree, sinks
├─ showSetupScreens() → trust, login, onboarding
└─ launchRepl() → <App><REPL/></App>
```

**Key insight**: Only 1 path leads to REPL. All 15+ fast-paths are utilities that execute and exit.

---

## Next: REPL Initialization Analysis

The normal path (no fast-path match) enters the full REPL system. Before the interactive loop starts, extensive initialization occurs:

### Pre-REPL Initialization (TODO - Next Analysis)
- `main.tsx` module evaluation: MDM prefetch, keychain prefetch, ~150 static imports
- `main()`: Commander.js program construction, option parsing
- `.action()`: Extract CLI options, build tool permission context, load tools
- `setup()`: cwd setting, hooks config, worktree creation, analytics sinks
- `showSetupScreens()`: trust dialog, OAuth login, onboarding flows
- Post-trust: telemetry initialization, MCP connections, bootstrap data

### Interactive Loop (TODO - After Initialization)
- `launchRepl()`: Render `<App><REPL/></App>`
- User prompt → `query()` async generator → API + tools → loop until done
- Query engine, tool system, background processes

The REPL depends heavily on the initialization phase - we should analyze that comprehensive setup process before diving into the interactive loop itself.
