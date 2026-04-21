# 13 — Interruption Handling and Error Recovery

**Scope**: How the Claude Code CLI reacts to disruptions — OS signals
(SIGINT, SIGTERM, SIGHUP), terminal orphaning, user-initiated in-UI
cancels (Escape, Ctrl+C), API-side transient failures (429/529,
auth expiry, connection drops), configuration errors on startup, and
background agent cancellation. The system has three distinct layers
that each handle cancellation differently — OS signals, in-process
`AbortController` trees, and domain-specific retry logic.

---

## 1. Three layers of cancellation

The code distinguishes three non-overlapping interruption modalities:

1. **Process-level** — POSIX signals (`SIGINT`, `SIGTERM`, `SIGHUP`)
   and orphan detection, handled in `src/utils/gracefulShutdown.ts`.
   This tier tears down the whole process.
2. **In-process abort** — `AbortController`s wired into the query
   loop and tool calls. The REPL owns the controller
   (`src/screens/REPL.tsx:865-869`), and escape/ctrl+C aborts it with
   a `reason` string that downstream code dispatches on.
3. **API retry/fallback** — `src/services/api/withRetry.ts`
   translates transient API errors (429, 529, auth failures, stale
   connections) into structured retries with exponential backoff,
   fast-mode cooldowns, model fallback, and persistent-mode
   heartbeating.

Crucially, an abort signal flows from (1) → (2) (signal handlers
invoke `gracefulShutdown`, which runs cleanup functions which have
already cascaded to controllers), and from (2) → (3) (the API
retry loop checks `signal.aborted` in hot paths).

---

## 2. Layer 1 — Signal handlers (`gracefulShutdown.ts`)

### 2.1 Registration — `setupGracefulShutdown()` (`gracefulShutdown.ts:237-334`)

Memoized setup function called from `init()`
(`src/entrypoints/init.ts:87`). Registers handlers for:

| Signal | Exit code | Platforms | Notes |
|---|---|---|---|
| `SIGINT` | `0` | all | Skipped in print mode — `print.ts` owns SIGINT there |
| `SIGTERM` | `143` (128+15) | all | Standard kill |
| `SIGHUP` | `129` (128+1) | non-Windows | Terminal hang-up |
| `uncaughtException` | — | all | `logEvent('tengu_uncaught_exception')` |
| `unhandledRejection` | — | all | `logEvent('tengu_unhandled_rejection')` |

All three signals funnel to `void gracefulShutdown(exitCode)`. The
"void" is deliberate — the signal handler cannot await, so it kicks
off the async shutdown and lets it race the failsafe timer.

### 2.2 Bun sigaction pin (`gracefulShutdown.ts:237-254`)

Documents a subtle Bun bug: any short-lived `signal-exit` v4
subscriber that unsubscribes when it's the last subscriber triggers
`v4.unload()` → `removeListener(sig, fn)` → *kernel* sigaction reset
to the default (terminate). This silently disables the handlers
registered above, so mid-session the next SIGTERM skips cleanup
entirely.

Fix: `onExit(() => {})` at line 254 — a no-op subscriber that's
never unsubscribed, keeping the emitter count > 0 so `unload()` never
runs. Harmless under Node.js.

### 2.3 Orphan detection (`gracefulShutdown.ts:281-296`)

macOS revokes the TTY file descriptor without delivering SIGHUP when
a terminal is forcibly closed. A 30-second `setInterval` polls
`process.stdout.writable && process.stdin.readable`; once false, it
calls `gracefulShutdown(129)`. `.unref()` keeps the interval from
holding the process alive.

This is one of the subtler error-recovery mechanisms in the codebase
— without it, an orphaned CLI would hang forever, blocking the
workstation on session resume.

### 2.4 `gracefulShutdown()` pipeline (`gracefulShutdown.ts:391-523`)

Ordered phases, each time-bounded:

1. **Re-entrancy guard** (`shutdownInProgress` flag) — second-press
   returns immediately.
2. **Failsafe timer** (`gracefulShutdown.ts:417-426`) — hard-caps the
   whole process at `max(5000, sessionEndTimeoutMs + 3500)` ms.
   Fires `cleanupTerminalModes()` + `printResumeHint()` + `forceExit(code)`
   so a stuck MCP connection can't leave the user at a black screen.
   The budget scales with the user-configured SessionEnd hook
   timeout (`getSessionEndHookTimeoutMs()` resolves this before
   arming; comment explicitly references gh-32712 follow-up).
3. **Terminal restoration** (first!) — `cleanupTerminalModes()`
   calls `writeSync(1, ...)` to disable: mouse tracking, alt-screen,
   extended key reporting (DECSET 2027/Kitty), focus events,
   bracketed paste, cursor show, iTerm2 progress, OSC tab status,
   terminal title. This runs *before* any async work so the user
   sees a sane prompt even if cleanup hangs.
4. **Resume hint printed** — `printResumeHint()` emits
   `claude --resume <sessionId or "Title">` on main buffer, dimmed
   chalk. Skipped if already printed or non-TTY.
5. **`runCleanupFunctions()` with 2s budget** — races the cleanup
   registry promises against `setTimeout(2000)`. Any registered
   cleanup (see §3) that hangs on dead I/O is dropped.
6. **`executeSessionEndHooks()` with its own budget** — wrapped in
   `AbortSignal.timeout(sessionEndTimeoutMs)`, so hook scripts can't
   exceed the configured limit.
7. **`profileReport()`** — startup-profile flush before analytics die.
8. **`tengu_cache_eviction_hint`** analytics event — signals the
   inference layer that this session's prompt cache can be evicted.
9. **Analytics flush capped at 500ms** — `Promise.race` against
   `sleep(500)` guards against 1P exporter hangs (10s axios POSTs
   that would eat the entire failsafe budget).
10. **`forceExit(code)`** (`gracefulShutdown.ts:193-232`) — tries
    `process.exit(code)`; if Bun throws EIO on dead stdout, falls
    back to `process.kill(pid, 'SIGKILL')`.

`gracefulShutdownSync()` at `gracefulShutdown.ts:336-359` is the
sync-caller-friendly variant: sets `process.exitCode`, kicks off the
async shutdown, and catches its rejection by force-exiting with the
terminal-cleanup + resume-hint fallback.

---

## 3. Layer 1.5 — Cleanup registry (`cleanupRegistry.ts`)

A minimal pub-sub:

```ts
const cleanupFunctions = new Set<() => Promise<void>>()
registerCleanup(fn): () => void   // returns unregister
runCleanupFunctions(): Promise<void>  // Promise.all
```

Only 25 lines. Deliberately separate from `gracefulShutdown.ts` to
avoid circular imports (every cleanup-holder imports this module
but shouldn't drag in terminal/ink/analytics).

Cleanups registered at init (`src/entrypoints/init.ts`):

- `shutdownLspServerManager` (line 189)
- `cleanupSessionTeams` — swarm team files (line 195-199)

Cleanups registered during setup/runtime (grep for `registerCleanup(`):

- `FileChanged` watcher disposal
- Plugin hook hot-reload watcher teardown
- Remote bridge disconnection
- Native installer pidLock release

All 25 or so handlers run in parallel under `Promise.all`, bounded
by the 2-second race in `gracefulShutdown()`.

---

## 4. Layer 2 — In-process abort (the `AbortController` tree)

### 4.1 REPL owns the top-level controller

`src/screens/REPL.tsx:865-869`:

```ts
const [abortController, setAbortController] = useState<AbortController | null>(null);
const abortControllerRef = useRef<AbortController | null>(null);
abortControllerRef.current = abortController;
```

The controller is created per-turn
(`main.tsx:1882`, `1986`: `new AbortController()`) and passed through
`onQuery()` → `onQueryImpl()` → `toolUseContext` → every tool call
→ every API request. The ref mirror exists because effects fire
stale closures.

### 4.2 `createAbortController` + `createChildAbortController`

`src/utils/abortController.ts` provides the two factories:

- `createAbortController(maxListeners = 50)` — sets
  `setMaxListeners(50, controller.signal)` to suppress MaxListeners
  warnings when many tools wire into the same signal.
- `createChildAbortController(parent)` — wires child to parent via
  **WeakRef** on both sides (lines 80-96). Memory-critical: a
  parent that accumulates child listeners for long-running sessions
  would leak. WeakRef + `{ once: true }` means abandoned children
  can still be GC'd, and the parent-listener cleanup handler
  auto-removes when the child aborts (whether directly or from the
  parent cascade).

### 4.3 `createCombinedAbortSignal`

`src/utils/combinedAbortSignal.ts` composes `signalA`, optional
`signalB`, and an optional `timeoutMs`. The module comment calls out
a Bun-specific bug: `AbortSignal.timeout()` timers are finalized
lazily and hold ~2.4KB each in native memory until they fire. This
implementation uses plain `setTimeout`/`clearTimeout` so the timer
is freed immediately on `cleanup()`.

### 4.4 Abort reasons are dispatched semantically

`abortController.abort(reason)` is called with a *string reason* so
downstream code can distinguish **why**:

| Reason | Call site | Handling |
|---|---|---|
| `'user-cancel'` | `REPL.tsx:2147, 2152` | Escape / Ctrl+C during permission prompt or mid-query |
| `'background'` | `REPL.tsx:2528` | Ctrl+B foreground→background transition |
| `'interrupt'` | `REPL.tsx:4102` | `'now'`-priority queued command (UDS inject, remote control) |

The reason surfaces in `signal.reason` and is checked at
`REPL.tsx:3010` to decide whether to preserve or discard in-flight
state:

```ts
if (abortController.signal.reason === 'user-cancel' && !queryGuard.isActive
    && inputValueRef.current === '' && getCommandQueueLength() === 0
    && !store.getState().viewingAgentTaskId) { ... }
```

### 4.5 `isAbortError(e)` detector (`utils/errors.ts:27-33`)

Checks three distinct abort-shaped errors:

```ts
return e instanceof AbortError
    || e instanceof APIUserAbortError           // SDK class (minified!)
    || (e instanceof Error && e.name === 'AbortError')  // DOMException
```

Comment is important: `constructor.name` doesn't work for SDK
classes because minified builds mangle names to things like `nJT`
and the SDK never sets `this.name`, so string matching silently
fails in production. All abort detection in the codebase routes
through this helper.

---

## 5. Layer 2 — User-facing cancel UI

### 5.1 `CancelRequestHandler` (`hooks/useCancelRequest.ts`)

Rendered inside `KeybindingSetup`. Returns `null` — it only
registers keybinding handlers. Three keybindings:

- **`chat:cancel`** (Escape) — active when there's an in-flight
  abort signal OR queued commands, AND no overlay/special context
  is claiming escape.
- **`app:interrupt`** (Ctrl+C) — active in the same contexts as
  above, plus when viewing a teammate transcript. When viewing a
  teammate, it kills all background agents AND exits the teammate
  view.
- **`chat:killAgents`** (Ctrl+X Ctrl+K, configurable) — always
  active because the chord prefix is always consumed, but the
  handler gates internally. First press shows a confirmation hint;
  second press within 3s (`KILL_AGENTS_CONFIRM_WINDOW_MS`) kills
  all running local agent tasks.

### 5.2 Cancel priority order (`useCancelRequest.ts:87-115`)

```
Priority 1: active task running → onCancel() (abort + drain queue)
Priority 2: no task but queued commands → popCommandFromQueue()
Fallback:   onCancel()
```

Priority 1 takes precedence over queue management — "users can
always interrupt Claude."

### 5.3 Context guards (the `isContextActive` predicate)

Escape is suppressed when any of these own it:
`transcript` screen, history-search, message-selector, local JSX
command, help dialog, any overlay, or vim INSERT mode. Ctrl+C
retains wider applicability (always cancels the request) but
respects `isViewingTeammate`.

### 5.4 `onCancel()` in REPL (`REPL.tsx:2125-2163`)

Five things happen:

1. Flush streaming text to messages buffer.
2. `resetLoadingState()`.
3. Clear token budget snapshot (`TOKEN_BUDGET` feature).
4. Dispatch on active focus:
   - `focusedInputDialog === 'tool-permission'` → call
     `toolUseConfirmQueue[0]?.onAbort()`.
   - `focusedInputDialog === 'prompt'` → reject all pending
     prompts with `new Error('Prompt cancelled by user')`, then
     abort.
   - Remote mode → `activeRemote.cancelRequest()` (sends interrupt
     to CCR).
   - Default → `abortController?.abort('user-cancel')`.
5. **Null the controller** (`setAbortController(null)`) so
   subsequent Escape presses see a fresh state, not a stale-aborted
   signal.

Line 2162: `void mrOnTurnComplete(messagesRef.current, true)` fires
directly because `forceEnd()` skipped the finally path in query.

---

## 6. Layer 3 — API retry and fallback (`services/api/withRetry.ts`)

### 6.1 Shape: `async function* withRetry<T>(...)`

An async generator that yields `SystemAPIErrorMessage` objects on
each retry (so the REPL can display "Retrying in 5s…" to the user)
and returns `T` on final success. Throws `CannotRetryError` (wrapped
original) or `FallbackTriggeredError` as terminal failure modes.

### 6.2 Retryable conditions (`shouldRetry` at `withRetry.ts:696-787`)

Retries for:

- Any status ≥ 500 (server errors).
- 408 (request timeout), 409 (lock timeout).
- 429 for non-subscribers (or Enterprise subscribers — subscribers
  get long waits and should fail fast).
- 401 (clears API key helper cache + refreshes OAuth token upstream).
- 403 "OAuth token has been revoked" (same as 401).
- 529 overloaded (checked in message even if status code is dropped
  by SDK during streaming).
- `APIConnectionError` (unconditional).
- `x-should-retry: true` header from server.
- In CCR mode (`CLAUDE_CODE_REMOTE`), 401/403 are retried
  unconditionally — the JWT infra provides tokens, so a blip
  isn't bad credentials.

Never retries: mock rate-limit errors (injected by
`/mock-limits` for testing), or when
`x-should-retry: false` (except ants on 5xx).

### 6.3 Persistent mode (`CLAUDE_CODE_UNATTENDED_RETRY`)

Ant-only feature: `withRetry.ts:100-104`. When enabled, 429/529
retry **indefinitely** with exponential backoff capped at 5
minutes. Long sleeps (> 60s) are chunked at 30s
(`HEARTBEAT_INTERVAL_MS`) with a yielded `SystemAPIErrorMessage`
each tick — this is the keep-alive channel that prevents the host
environment from marking the session idle mid-wait. The `attempt`
counter is clamped at `maxRetries+1` so the for-loop never
terminates; `persistentAttempt` grows independently for telemetry.

Comment at line 92-95 notes this is a stopgap pending ANT-344's
dedicated keep-alive channel.

### 6.4 Fast-mode fallback

If `fastMode` is active and 429/529 fires:

- **Short retry-after** (< 20s, `SHORT_RETRY_THRESHOLD_MS`) →
  wait + retry with fast mode still on (preserves prompt cache).
- **Long / unknown retry-after** →
  `triggerFastModeCooldown(Date.now() + cooldownMs, reason)` where
  `reason` is `'overloaded'` or `'rate_limit'`. Switches model
  back to standard speed for `max(retryAfter, MIN_COOLDOWN_MS=10min)`.
- **Overage exhaustion** (`anthropic-ratelimit-unified-overage-disabled-reason`
  header) → `handleFastModeOverageRejection(reason)` permanently
  disables fast mode with a specific user-facing message.

### 6.5 Opus fallback chain

`withRetry.ts:326-365`: after `MAX_529_RETRIES=3` consecutive 529s
on a primary Opus model (or any primary model if
`FALLBACK_FOR_ALL_PRIMARY_MODELS` is set):

- If `options.fallbackModel` is set → throw
  `FallbackTriggeredError(options.model, options.fallbackModel)`
  and let the caller swap models.
- For external (non-ant, non-sandbox, non-persistent) users → throw
  `CannotRetryError(REPEATED_529_ERROR_MESSAGE)` so the user sees
  a specific "API is overloaded, please try again later" message.

### 6.6 Foreground vs background 529 triage

`FOREGROUND_529_RETRY_SOURCES` set at `withRetry.ts:62-82`. During
a capacity cascade, retrying background calls (summaries, titles,
suggestions, classifiers) amplifies gateway load 3-10× for no user
benefit. So only foreground sources (`repl_main_thread`, `sdk`,
`agent:*`, `compact`, `hook_*`, `verification_agent`,
`side_question`, `auto_mode`, `bash_classifier` under BASH_CLASSIFIER
gate) retry 529; everything else throws `CannotRetryError`
immediately. Telemetry: `tengu_api_529_background_dropped`.

### 6.7 Context-window overflow self-healing

`parseMaxTokensContextOverflowError` at `withRetry.ts:550-595` parses
the API's error message (`"input length and \`max_tokens\` exceed
context limit: X + Y > Z"`), computes a safe `availableContext =
contextLimit - inputTokens - 1000`, and sets
`retryContext.maxTokensOverride = max(FLOOR_OUTPUT_TOKENS=3000,
availableContext, minRequired)`. The next attempt re-issues the
request with the lower `max_tokens`. Noted as legacy fallback — the
extended-context-window beta returns a `model_context_window_exceeded`
stop_reason instead, handled elsewhere.

### 6.8 Stale connection (ECONNRESET / EPIPE)

Gated on `tengu_disable_keepalive_on_econnreset` feature. When
detected, `disableKeepAlive()` is called and the next retry builds
a fresh HTTP agent. Client is re-fetched via `getClient()` so a new
dispatcher takes over.

### 6.9 Cloud-provider auth recovery

- Bedrock: 403 or `AwsCredentialsProviderError` →
  `clearAwsCredentialsCache()` + retry.
- Vertex: `google-auth-library` "Could not load / refresh
  credentials" / "invalid_grant" OR 401 →
  `clearGcpCredentialsCache()` + retry.
- OAuth 401 or "token revoked" 403 → `handleOAuth401Error(token)` +
  force `getClient()` refresh.
- API key: 401 on first-party API → `clearApiKeyHelperCache()`.

---

## 7. Startup configuration-error recovery

`src/entrypoints/init.ts:215-236` wraps the entire init flow in
`try/catch` specifically to catch `ConfigParseError`:

- **Non-interactive**: write the error to stderr and
  `gracefulShutdownSync(1)`. The dialog can't render reliably in
  JSON consumers (the comment cites desktop marketplace plugin
  manager running `plugin marketplace list --json` in a VM sandbox).
- **Interactive**: dynamic import of
  `components/InvalidConfigDialog.js` and render it. Dialog handles
  its own exit.

Any non-`ConfigParseError` rethrows (becomes uncaughtException →
logged via `tengu_uncaught_exception`).

---

## 8. Control-flow diagram

```
                             [OS signal arrives]
                                    │
                          ┌─────────┴─────────┐
                          │                   │
                       SIGINT              SIGTERM/SIGHUP
                          │                   │
                 (print-mode? return)         │
                          │                   │
                          └────────┬──────────┘
                                   ▼
                     gracefulShutdown(exitCode)
                                   │
                    re-entrancy guard (shutdownInProgress)
                                   │
                   arm failsafe timer (max 5s, or SessionEnd+3.5s)
                                   │
                   cleanupTerminalModes()  ← drop mouse tracking,
                                                alt screen, Kitty KB,
                                                title, tab status
                                   │
                   printResumeHint()  ← main buffer, dim chalk
                                   │
                   runCleanupFunctions()  ── 2s race ──┐
                                   │                   │
                   executeSessionEndHooks()  ── budget ┤
                                   │                   │
                   profileReport()                     │
                                   │                   │
                   tengu_cache_eviction_hint            │
                                   │                   │
                   analytics flush  ── 500ms race ──┐   │
                                   │               │   │
                                   ▼               ▼   ▼
                              forceExit(code)
                                   │
                          try process.exit
                          catch → process.kill(SIGKILL)
```

User-initiated cancellation:

```
Escape or Ctrl+C
       │
CancelRequestHandler.handleCancel / handleInterrupt
       │
onCancel() in REPL.tsx:2125
       │
  ┌────┼────┬────────────────┬─────────────┐
  │    │    │                │             │
tool- prompt  remote-mode   default      null-controller
permission  queue reject   cancelRequest  abort('user-cancel')
  │    │    │                │             │
  └────┴────┴────────────────┴─────────────┘
       │
       ▼
  abortController propagates to:
    - Every tool call in-flight
    - Every API request via signal param
    - Every registered child controller (WeakRef)
    - withRetry loop: checks signal.aborted each iteration
```

---

## 9. Integration points with other subsystems

- **Graceful-shutdown ↔ hooks** — `executeSessionEndHooks()` is
  called inside `gracefulShutdown()`. The hook budget
  (`CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`, default 1.5s) is
  resolved *before* the failsafe timer arms so the failsafe scales
  with it.
- **Graceful-shutdown ↔ cleanup registry** — `runCleanupFunctions()`
  runs all registered cleanups in parallel with a 2s budget.
- **Graceful-shutdown ↔ Ink** — `cleanupTerminalModes()` calls
  `instances.get(process.stdout)?.unmount()` to do the final
  alt-screen exit exactly once, avoiding double-DECRC that would
  clobber the resume hint (gh-16xxx range; see file comment block
  at lines 76-106).
- **AbortController ↔ tool execution** — Every tool receives the
  signal via `toolUseContext.abortSignal`; tools that spawn
  subprocesses are responsible for propagating cancellation (see
  `ShellCommand.ts`, `execFileNoThrow.ts`).
- **AbortController ↔ query loop** — `src/query.ts` and
  `QueryEngine.ts` check `signal.aborted` at multiple choke points
  so a turn can end between tool calls cleanly.
- **withRetry ↔ signal** — Loop header checks `signal?.aborted`
  and throws `APIUserAbortError()` so the SDK's own abort semantics
  take over. `sleep(delayMs, signal, { abortError })` aborts
  mid-sleep.
- **withRetry ↔ fastMode** — Separate module (`utils/fastMode.ts`)
  holds cooldown state; retry loop reads/writes via
  `triggerFastModeCooldown`, `isFastModeCooldown`,
  `isFastModeEnabled`.
- **User interrupt ↔ tool permission UI** — `onCancel` calls
  `toolUseConfirmQueue[0]?.onAbort()` to let the permission prompt
  short-circuit with the user's rejection semantics.
- **User interrupt ↔ remote mode** — CCR has its own interrupt
  channel (`activeRemote.cancelRequest()`) that's routed through
  the remote socket instead of the local abort controller.
- **Background-session cancellation** — `abortController.abort('background')`
  at `REPL.tsx:2528` is distinguishable from `'user-cancel'` so the
  post-abort path *doesn't* display "cancelled" to the user; it
  hands off to a background session instead.
- **Conversation recovery** — `src/utils/conversationRecovery.ts`
  handles `--resume` and `--continue`, loading the last session
  from JSONL. This is *not* part of error-recovery proper (it runs
  on startup, not response to failure), but is the mechanism by
  which a crashed or force-killed session can be recovered on next
  launch.

---

## 10. Key observations and flagged complexities

### Sophistication level: **high**

The interruption system has at least 7 distinct recovery mechanisms
layered on top of each other:

1. Signal-handler → async-graceful-shutdown race
2. Bun sigaction-pin workaround for signal-exit interaction
3. macOS orphan detection (30s poll for `stdout.writable`)
4. Three-layer AbortController tree with WeakRef memory safety
5. Semantic abort reasons (`user-cancel` / `background` / `interrupt`)
6. API retry with foreground/background source triage and
   persistent-mode heartbeat yields
7. Provider-specific auth recovery (Bedrock, Vertex, OAuth, API key)

Each layer has at least one documented bug-fix comment pointing to
a specific production incident (see §2.2 Bun sigaction, §6.2 mock
rate limit, §6.7 max_tokens overflow, `gracefulShutdown.ts:76-106`
Ink DECRC double-unmount).

### Follow-up deep dives recommended

- **Rate-limit UX flow** — `src/services/rateLimitMessages.ts` and
  the `SystemAPIErrorMessage` rendering path in the REPL are
  surface-level here; the actual UI/telemetry coupling deserves
  its own pass.
- **Tool-level cancellation** — `tools/ShellCommand.ts` and
  `tools/computerUse/*` have non-trivial cancellation semantics
  (killing subprocess trees, un-wedging stuck pagers) that
  aren't covered here.
- **Permission-prompt `onAbort`** — The `ToolUseConfirm` lifecycle
  (how permission prompts handle abort vs reject vs timeout) is a
  separate sub-flow.
- **Session backgrounding** (`useSessionBackgrounding`) — Ctrl+B
  lifecycle and how messages/queries hand off to a background
  session without interruption appears to be a major feature.
- **Persistent-retry keep-alive mechanism** — ANT-344 TODO flags
  this as stopgap; a dedicated channel would be a separate ticket.
- **`conversationRecovery.ts`** — full scope of session replay +
  reopening is a separate task (#6 in this plan).

---

## 11. Code references quick index

| Concern | File:line |
|---|---|
| Signal registration | `src/utils/gracefulShutdown.ts:237-334` |
| Bun sigaction pin | `src/utils/gracefulShutdown.ts:237-254` |
| Orphan detection | `src/utils/gracefulShutdown.ts:281-296` |
| gracefulShutdown pipeline | `src/utils/gracefulShutdown.ts:391-523` |
| forceExit with SIGKILL fallback | `src/utils/gracefulShutdown.ts:193-232` |
| cleanupTerminalModes | `src/utils/gracefulShutdown.ts:59-136` |
| printResumeHint | `src/utils/gracefulShutdown.ts:144-184` |
| gracefulShutdownSync | `src/utils/gracefulShutdown.ts:336-359` |
| runCleanupFunctions | `src/utils/cleanupRegistry.ts:23-25` |
| createAbortController | `src/utils/abortController.ts:16-22` |
| createChildAbortController (WeakRef) | `src/utils/abortController.ts:68-99` |
| createCombinedAbortSignal | `src/utils/combinedAbortSignal.ts:15-47` |
| isAbortError | `src/utils/errors.ts:27-33` |
| REPL abort ownership | `src/screens/REPL.tsx:865-869` |
| REPL onCancel | `src/screens/REPL.tsx:2125-2163` |
| abort('background') for Ctrl+B | `src/screens/REPL.tsx:2528` |
| abort('interrupt') for 'now' msg | `src/screens/REPL.tsx:4102` |
| CancelRequestHandler | `src/hooks/useCancelRequest.ts` |
| Cancel priority order | `src/hooks/useCancelRequest.ts:87-115` |
| chat:cancel keybind | `src/hooks/useCancelRequest.ts:164-167` |
| app:interrupt keybind | `src/hooks/useCancelRequest.ts:217-220` |
| chat:killAgents two-press | `src/hooks/useCancelRequest.ts:225-273` |
| withRetry main loop | `src/services/api/withRetry.ts:170-517` |
| shouldRetry logic | `src/services/api/withRetry.ts:696-787` |
| FOREGROUND_529_RETRY_SOURCES | `src/services/api/withRetry.ts:62-82` |
| Persistent retry / heartbeat | `src/services/api/withRetry.ts:477-506` |
| Fast-mode fallback | `src/services/api/withRetry.ts:267-314` |
| Opus → fallbackModel | `src/services/api/withRetry.ts:326-365` |
| parseMaxTokensContextOverflowError | `src/services/api/withRetry.ts:550-595` |
| Stale-connection detection | `src/services/api/withRetry.ts:112-118, 218-230` |
| Bedrock/Vertex auth recovery | `src/services/api/withRetry.ts:631-694` |
| ConfigParseError recovery | `src/entrypoints/init.ts:215-236` |
| Error classes (ClaudeError/AbortError) | `src/utils/errors.ts:3-71` |
| TelemetrySafeError | `src/utils/errors.ts:93-100+` |
