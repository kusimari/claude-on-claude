# 06 — `REPL.tsx` and User Input Handling

**Scope**: The top-level interactive screen component
`src/screens/REPL.tsx` (≈5000 LOC) and the input-submission pipeline
that turns a keystroke into a model query. Covers:

1. REPL component structure and state model.
2. `PromptInput` → `onSubmit` → `handlePromptSubmit` → `executeUserInput`
   → `processUserInput` → `onQuery` pipeline.
3. `QueryGuard` state machine (replaces the legacy dual-state
   `isLoading`/`isQueryRunning`).
4. Message-queue / immediate-command fan-out.
5. Ink-framework patterns specific to this file (and a flag: **yes**,
   those patterns need dedicated explanation for reviewers).
6. Integration with remote/bridge/ssh session paths, cancel flow, and
   scheduled-task/teammate prompts.

Line references are to `src/screens/REPL.tsx` unless noted.

---

## 1. Component layout and render tree

`REPL` is a single function component (`REPL.tsx:324`) that roughly
owns **every** stateful concern of a session's interactive shell:
messages, tools, MCP clients, commands, input buffer, streaming text,
spinner timing, permission dialogs, cancel flow, bridge/SSH/direct
connect, worktree display, fullscreen ScrollBox ref, etc. The full
prop surface is at `REPL.tsx:332-473` (≈150 props).

The render output has two branches:

- **Transcript mode** (`REPL.tsx:4406-4489`) — a read-only historical
  view triggered by ctrl+o / `--transcript`. Renders
  `<Messages>` inside `KeybindingSetup` with `ScrollKeybindingHandler`,
  `CancelRequestHandler`, and `FullscreenLayout` (if fullscreen).
- **Main mode** (`REPL.tsx:4548-4998`) — the live REPL. Wraps a
  `<MCPConnectionManager>` around `<FullscreenLayout>` with:
  - `scrollable` slot: header → `<Messages>` → `<TungstenLiveMonitor>`
    (ant) → `<SpinnerWithVerb>` or `<BriefIdleStatus>` →
    `<PromptInputQueuedCommands>` (fullscreen only).
  - `bottom` slot: permission sticky footer, immediate local-jsx
    command JSX, `<TaskListV2>`, `SandboxPermissionRequest`,
    `PromptDialog`, `ElicitationDialog`, `CostThresholdDialog`,
    `IdleReturnDialog`, `IdeOnboardingDialog`, `AntModelSwitchCallout`,
    `RemoteCallout`, `PluginHintMenu`, `LspRecommendationMenu`,
    `UltraplanChoiceDialog`, `UltraplanLaunchDialog`, `FeedbackSurvey`,
    `AutoRunIssueNotification`, and finally the `<PromptInput>` itself
    (`REPL.tsx:4903-4905`).

The decision about *which* dialog is focused is centralized in
`focusedInputDialog` (a derived value), with modal overlays routed
through `useIsOverlayActive` / `useRegisterOverlay`. A single
`<PromptInput>` renders only when no other dialog owns the input
(`REPL.tsx:4894`).

Both branches are optionally wrapped in `<AlternateScreen>`
(`REPL.tsx:4485-4488`, `4999-5003`) — the alt-buffer is entered only
when fullscreen is enabled, and must stay consistently entered across
transcript/main toggles (same root type + props for reconciliation).

---

## 2. Key state and the `QueryGuard` state machine

### 2.1 `QueryGuard` — the single source of truth for "is the REPL busy?"

Historically the REPL carried two flags (React `isLoading` state and a
`isQueryRunning` ref) that could desync because React state is
batched/async while refs are synchronous. `REPL.tsx:897-900` creates a
single `QueryGuard` instance per REPL lifetime. It exposes:

- `reserve()` — transitions `idle → dispatching`. Called from
  `executeUserInput` **before** the first await (see
  `handlePromptSubmit.ts:437`) so that concurrent submissions from
  other call sites see `isActive === true` and enqueue.
- `tryStart()` — atomic `idle|dispatching → running`; returns a
  generation number or `null` if already running
  (`REPL.tsx:2869-2886`).
- `end(generation)` — `running → idle` but **only if generation
  matches**, so a late-firing `finally` from a stale generation
  cannot clobber a newer query (see `REPL.tsx:2923` in the finally
  block).
- `forceEnd()` — bumps the generation and forces idle. Only used by
  `onCancel` so the auto-restore logic below can differentiate
  user-cancel from a clean end (`REPL.tsx:2118`).
- `cancelReservation()` — `dispatching → idle`, no-op if `running`.
  Called from `handlePromptSubmit`'s finally as a safety net
  (`handlePromptSubmit.ts:603`).
- `subscribe` / `getSnapshot` — `useSyncExternalStore` hooks
  (`REPL.tsx:904`).

The derived flags:

```ts
const isQueryActive     = useSyncExternalStore(queryGuard.subscribe,
                                               queryGuard.getSnapshot)
const [isExternalLoading, setIsExternalLoadingRaw] = useState(...)
const isLoading = isQueryActive || isExternalLoading
```

`isExternalLoading` (`REPL.tsx:911`) exists only for things that do
*not* route through `onQuery`: `useRemoteSession`, `useDirectConnect`,
`useSSHSession`, `useSessionBackgrounding`. Its setter is wrapped to
also `resetTimingRefs()` on true→ transition so the spinner doesn't
render "56 years elapsed" from `Date.now() - 0`
(`REPL.tsx:960-963`). The same guard for local queries lives inline
at `REPL.tsx:950-953`: on the first render where `isQueryActive` flips
true, timing refs are reset before the spinner reads them. INC-4549
motivated both — the comment is worth reading in place.

### 2.2 Major state

| Concern | Type | Where | Notes |
|---|---|---|---|
| Messages | `MessageType[]` | `REPL.tsx:1182` | `rawSetMessages` wrapped by `setMessages` (`1198-1222`) keeps `messagesRef` in sync *synchronously* — Zustand-style "ref is truth, React is projection". |
| Abort controller | `AbortController \| null` | `865-869` | Mirror ref `abortControllerRef` so useReplBridge can abort from a WS callback without a re-render. |
| Input value | `string` | `1331` | Lazy-init via `consumeEarlyInput()` so keystrokes received before React hydrates aren't lost. Mirrored in `inputValueRef` (`1332`). |
| Input mode | `'prompt' \| 'bash' \| ...` | `1372` | `bash`, `thinking`, `background` etc. See `types/textInputTypes.ts`. |
| Paste contents | `Record<number, PastedContent>` | `1423` | Images + long-paste texts; referenced by `[Image #N]` / `[Pasted text #N]` placeholders in the buffer. |
| Stream mode | `'requesting' \| 'responding' \| 'tool-use'` | `838` | Mirror ref `streamModeRef` (`847-848`) so `onSubmit`'s deps don't churn (flips ~10×/turn). |
| Streaming text | `string \| null` | `1461` | Cleared atomically when the assistant message lands — `visibleStreamingText` further trims to the last newline for line-by-line flow. |
| Queued commands | `QueuedCommand[]` | module-level store (`messageQueueManager.ts`) | Exposed here via `useSyncExternalStore` in `useQueueProcessor` (`useQueueProcessor.ts:40-46`). |
| Tool JSX overlay | `{ jsx, shouldHidePromptInput, isLocalJSXCommand, isImmediate }` | `1032` | `setToolJSX` wrapper (`1060-1100`) preserves a local-JSX command unless `clearLocalJSX: true` is explicitly set, so tool output can't steal the UI from a `/config` dialog. |
| Tool use confirm queue | `ToolUseConfirm[]` | `1101` | FIFO queue rendered via `PermissionRequest` (`4519`). |
| Prompt queue (hook) | `PromptRequest[]` | `1110` | Backs `<PromptDialog>` for `UserPrompt`-type hooks. |
| `userInputOnProcessing` | `string \| undefined` | `920` | The placeholder echoed while the real user message hasn't hit `setMessages` yet; hidden once `displayedMessages.length > userInputBaselineRef.current`. |
| Spinner timing | refs | `932-939` | `loadingStartTimeRef`, `totalPausedMsRef`, `pauseStartTimeRef` feed `<SpinnerWithVerb>` via refs (no re-render on every tick). |

The big "why refs over state" theme recurs: the REPL renders ~30×/turn
with huge prop surfaces, and `useCallback` closures that close over
message-array-of-the-render get pinned into every `MessageRow` fiber.
Refs keep callbacks stable and let long sessions GC old render scopes
(see `onSubmitRef` at `3613`, with an ~35MB savings comment).

---

## 3. The submission pipeline

### 3.1 End-to-end path

The input pipeline has several entry points but converges on
`executeUserInput` inside `handlePromptSubmit.ts`:

```
keystroke (terminal)
  ↓ ink.useInput() in <TextInput> / <VimTextInput>
  ↓ 'return' key → TextInput.props.onSubmit(value)  (PromptInput.tsx)
  ↓ PromptInput.onSubmit()  (PromptInput.tsx:984) — suggestion acceptance,
    directory completion, @name routing, agent-view routing,
    speculation acceptance
  ↓ onSubmitProp(input, { setCursorOffset, clearBuffer, resetHistory })
  ↓ REPL.onSubmit  (REPL.tsx:3142) — handles:
    - immediate local-jsx slash commands (while busy)
    - idle-return dialog gate
    - history push, stash restore, clear input
    - speculation accept → handleSpeculationAccept → onQuery
    - remote-mode: createUserMessage + activeRemote.sendMessage
    - otherwise ↓
  ↓ awaitPendingHooks()  (SessionStart hook deferred messages)
  ↓ handlePromptSubmit({ … })  (utils/handlePromptSubmit.ts:120)
     ↓ [exit commands] / [local-jsx immediate path while busy]
     ↓ if queryGuard.isActive || isExternalLoading:
         enqueue({ value, mode, pastedContents, … })  → return
       else:
         executeUserInput([cmd])  (handlePromptSubmit.ts:396)
            ↓ createAbortController() + setAbortController
            ↓ queryGuard.reserve()         (idle → dispatching)
            ↓ runWithWorkload(turnWorkload, async () => {
                for each queued cmd:
                  await processUserInput({…})   → result.messages
                await onQuery(newMessages, abortController, shouldQuery,
                              allowedTools, model, onBeforeQuery, input, effort)
              })
            ↓ finally: queryGuard.cancelReservation()  (no-op if running)
            ↓ finally: setUserInputOnProcessing(undefined)
  ↓ onQuery  (REPL.tsx:2855)
     ↓ queryGuard.tryStart() — atomic dispatching→running, returns gen or null
     ↓ setMessages(prev => [...prev, ...newMessages])
     ↓ resetTimingRefs(), apiMetricsRef.current = []
     ↓ mrOnBeforeQuery + onBeforeQueryCallback
     ↓ onQueryImpl  (REPL.tsx:2661)
         ↓ getToolUseContext + getSystemPrompt + getUserContext + getSystemContext
         ↓ buildEffectiveSystemPrompt
         ↓ for await (const event of query({…}))
             → onQueryEvent → handleMessageFromStream → setMessages
     ↓ finally:
         queryGuard.end(generation)
         setLastQueryCompletionTime
         resetLoadingState + mrOnTurnComplete + sendBridgeResult
         Auto-restore on 'user-cancel' (REPL.tsx:3010-3022)
```

### 3.2 Dispatch modes inside `REPL.onSubmit`

`REPL.onSubmit` (`REPL.tsx:3142-3545`) is the load-bearing callback
and handles many orthogonal concerns in a fixed order:

1. **`repinScroll()`** — snap to bottom on every submit.
2. **Resume proactive mode** if paused (KAIROS / PROACTIVE build).
3. **Immediate slash-command fast path** (`3161-3282`) —
   when busy and `cmd.immediate === true` (or keybinding-triggered),
   execute the command's `call(onDone, context, args)` directly with
   an overlay JSX registered via `setToolJSX({ isLocalJSXCommand: true })`.
   Used for `/btw`, `/sandbox`, `/assistant`, `/issue`, `/config`
   (while loading). Returns early so nothing is queued.
4. **Idle-return dialog gate** (`3292-3310`) — if the user has been
   idle beyond `CLAUDE_CODE_IDLE_THRESHOLD_MINUTES` and tokens exceed
   `CLAUDE_CODE_IDLE_TOKEN_THRESHOLD`, stash the input and show
   `<IdleReturnDialog>` instead of submitting.
5. **History push** (`3316-3326`) — with mode character prepended;
   bash mode also updates shell history cache.
6. **Stash restore vs defer** (`3328-3357`) — the submit path
   decides whether to restore a prior stashed prompt inline or wait
   for `await handlePromptSubmit` to return first; slash commands
   defer because pickers hide the input.
7. **Common UI cleanup** (`3358-3389`) when submitting-now: clear
   input, reset input mode, bump `submitCount`, `clearBuffer()`,
   reset `tipPickedThisTurnRef`, show the placeholder
   (`userInputOnProcessing`), increment attribution prompt count.
8. **Speculation acceptance** (`3391-3406`) — `handleSpeculationAccept`
   may inject partial messages and, if `queryRequired`, kicks off an
   empty `onQuery([], …)` so the speculative prefix becomes the
   turn's input without a user-visible re-typing.
9. **Remote-mode short-circuit** (`3408-3486`) — builds
   `ContentBlockParam[]` (text + image) from `pastedContents`, calls
   `createUserMessage` + `setMessages`, then
   `activeRemote.sendMessage`. Local slash commands of
   `type === 'local-jsx'` fall through so they can render in this
   process.
10. **`await awaitPendingHooks()`** (`3489`) — drains
    `SessionStart` hooks into the transcript before the first API
    call. The initial-message effect does the same hoist at `3105`
    because it bypasses `onSubmit`.
11. **`await handlePromptSubmit({…})`** (`3490-3519`) — the actual
    enqueue-or-execute dispatch.
12. **Deferred stash restore** (`3527-3532`) if the input was queued
    or routed to a slash command.

### 3.3 `handlePromptSubmit` (utils)

`src/utils/handlePromptSubmit.ts:120-387` has four branches:

1. **Queue-processor path** (`150-172`) — `queuedCommands?.length` is
   already-validated. Start a profiler block and call
   `executeUserInput` directly. This is how
   `useQueueProcessor` → `executeQueuedInput` fires (no input
   validation needed — commands were validated on enqueue).
2. **Exit commands** (`193-211`) — "`exit`", `":q"`, `":wq"`, etc.
   recursively route through `/exit` for the feedback dialog.
3. **Immediate local-jsx while busy** (`227-311`) — same mechanism as
   step 3 above, but this path fires when
   `handlePromptSubmit` is entered with `isActive || isExternalLoading`
   and the command has `immediate: true`.
4. **Enqueue while busy** (`313-351`) — mode ∈ `{prompt, bash}` only;
   if `hasInterruptibleToolInProgress` (e.g. SleepTool), abort the
   current turn with reason `'interrupt'` before enqueueing;
   otherwise just `enqueue({value, preExpansionValue, mode,
   pastedContents, skipSlashCommands, uuid})` and clear the input.
5. **Idle dispatch** (`353-386`) — build a `QueuedCommand` and call
   `executeUserInput([cmd])`. This unifies the direct-user and
   queue-processor paths onto the same executor so image resizing
   (inside `processUserInput`) happens regardless of origin.

### 3.4 `executeUserInput`

`handlePromptSubmit.ts:396-610` is the core loop. Key details:

- **Fresh abort controller per turn** (`419-420`). `queryGuard`
  guarantees no concurrent call, so there is no prior controller to
  inherit.
- **`queryGuard.reserve()`** before processing (`437`). Any
  `handlePromptSubmit` that races in after this point sees
  `isActive === true` and enqueues instead of starting a second
  execution. The reservation is a no-op if legacy queue-processor
  already reserved.
- **`runWithWorkload(turnWorkload, async () => {…})`** (`472-596`) —
  wraps the whole turn in an `AsyncLocalStorage` slot so
  void-detached bg agents (AgentTool, `executeForkedSlashCommand`)
  inherit the right workload tag across every await. Any synchronous
  return would clobber a process-global slot at the first detached
  await, so ALS is load-bearing.
- **Per-command loop** (`473-522`): first command gets
  `pastedContents`, `ideSelection`, `setUserInputOnProcessing`; the
  rest pass `skipAttachments: true` to avoid duplicating turn-level
  context. Each command yields `result.messages` (appended to
  `newMessages`) and, for the first command, `shouldQuery`,
  `allowedTools`, `model`, `effort`, `nextInput`,
  `submitNextInput`.
- **File history snapshot** (`525-538`) — one snapshot per user
  message before `onQuery` runs, so a later `/rewind` can restore
  pre-turn file state.
- **Branch A: messages exist** (`541-571`) — `resetHistory()`, clear
  `toolJSX`, `await onQuery(…)` with the primary command's model
  (resolved through `resolveSkillModelOverride`), `onBeforeQuery`,
  `primaryInput`, `effort`.
- **Branch B: no messages** (`572-586`) — local slash commands that
  complete without producing user messages (`/model`, `/theme`, …).
  `queryGuard.cancelReservation()` **before** clearing `toolJSX` to
  avoid a spinner flash (formula: `(!toolJSX || showSpinner) && isLoading`
  flickers if the order is reversed).
- **Chain next input** (`589-595`) — `nextInput` + `submitNextInput`
  from the first command gets enqueued or typed back; used by
  commands like `/discover`.
- **Finally** (`597-609`) — `cancelReservation()` and
  `setUserInputOnProcessing(undefined)` as safety nets if
  `processUserInput` threw or `onQuery` was skipped.

### 3.5 `onQuery` / `onQueryImpl`

`REPL.tsx:2855-3024` / `2661-2854` together run one full API turn:

- **`onQuery`** guards concurrency via `queryGuard.tryStart()`. If
  another query already owns the guard, it logs
  `tengu_concurrent_onquery_detected` and enqueues the user-visible
  messages (filtering meta) so they rerun on the next idle cycle.
  Otherwise it resets timing refs, appends `newMessages`, runs
  `mrOnBeforeQuery` + optional `onBeforeQueryCallback`, and hands
  off to `onQueryImpl`.
- **`onQueryImpl`** builds a fresh tool context
  (`getToolUseContext(messagesIncludingNewMessages, newMessages,
  abortController, mainLoopModelParam)`), resolves
  `{defaultSystemPrompt, baseUserContext, systemContext}` in
  parallel, assembles `systemPrompt` via
  `buildEffectiveSystemPrompt`, and runs `for await (const event of
  query(…))` feeding each event to `onQueryEvent` (see below).
- **`onQueryEvent`** (`REPL.tsx:2584-2660`) delegates to
  `handleMessageFromStream` (in `utils/messages.ts`), which yields
  concrete messages, streaming content deltas, stream-mode changes,
  streaming tool-use updates, tombstones, thinking deltas, and
  metrics. It also dedupes *ephemeral* progress messages (Sleep/Bash
  per-second ticks collapse to one, otherwise a long-running sleep
  would bloat the transcript to 120MB).
- **Finally block in `onQuery`** (`2919-3023`) is where cancel/resume
  semantics live:
  - `queryGuard.end(thisGeneration)` — only succeeds if no newer
    query took over; this gates the rest of the cleanup.
  - `setLastQueryCompletionTime` feeds the idle-notification timer.
  - `resetLoadingState` clears spinner, streaming, metrics,
    `userInputOnProcessing`.
  - `mrOnTurnComplete` fires the MoreRight hook with aborted flag.
  - `sendBridgeResultRef.current()` tells mobile bridge clients the
    turn is done.
  - Budget info + turn-duration message (`>30s` or with budget).
  - **Auto-restore** (`3010-3022`) runs *outside* the `end()` guard
    (because `onCancel` uses `forceEnd()` which bumps generation) —
    if the abort reason is `'user-cancel'`, the guard is idle, the
    input field is empty, there are no queued commands, and the user
    isn't viewing a teammate, walk back to the last selectable user
    message and call `restoreMessageSyncRef.current(lastUserMsg)` so
    the user gets their prompt back verbatim. Also calls
    `removeLastFromHistory()` so arrow-up doesn't show it twice.

---

## 4. Entry points that skip `onSubmit`

Several call sites push prompts without going through
`PromptInput`'s `onSubmit`:

- **`handleIncomingPrompt`** (`REPL.tsx:3996-4019`) — for messages
  injected by `useInboxPoller` (teammate mail), `useMailboxBridge`,
  `useTaskListWatcher`, and `useProactive`. Returns `false` if a
  query is active or if user prompts are already queued — system
  messages never pre-empt human input. Bypasses `handlePromptSubmit`
  entirely and calls `onQuery([userMessage], …)` directly after
  building a `createUserMessage({content, isMeta})`.
- **`executeQueuedInput`** (`3861-3888`) — feeds `useQueueProcessor`;
  calls `handlePromptSubmit` with `queuedCommands` set and no UI
  helpers.
- **`initialMessage` effect** (`3029-3141`) — CLI `--print` args or
  plan-mode exit with context clear. Routes string content through
  `onSubmit` (so `UserPromptSubmit` hooks fire), but plan-content or
  image content bypasses to `onQuery` directly. `awaitPendingHooks`
  is hoisted inline here because the bypass path wouldn't otherwise
  drain them.
- **`handleAutoRunIssue`** (`3581-3591`) and
  **`handleOpenRateLimitOptions`** (`3615-3621`) — programmatic
  `onSubmit('/issue', …)` / `onSubmit('/rate-limit-options', …)`.
  The latter uses `onSubmitRef` to stay stable across re-renders
  (see the heap-accounting comment at `3608-3614`).
- **`'now' priority`** (`4100-4104`) — a queued command with
  `priority: 'now'` (e.g. a UDS chat client asking for immediate
  attention) calls `abortControllerRef.current?.abort('interrupt')`
  via effect.

---

## 5. Cancel flow

`src/hooks/useCancelRequest.ts` renders a `null` component that only
registers keybindings — **must** be rendered inside `KeybindingSetup`.
Three bindings:

- **`chat:cancel` (Esc)** → `handleCancel`:
  - Priority 1: if there's an active non-aborted signal, clear
    `toolUseConfirmQueue` and call `onCancel()`.
  - Priority 2: otherwise if `hasCommandsInQueue()`, pop the last
    queued command (`popCommandFromQueue` = REPL's
    `handleQueuedCommandOnCancel`, which restores the popped text
    into the input buffer).
  - Gated on `isContextActive` (not in transcript/history
    search/selector/local-jsx/help/overlay/vim-INSERT) and
    `!isInSpecialModeWithEmptyInput` and `!isViewingTeammate`.

- **`app:interrupt` (ctrl+c)** → `handleInterrupt`:
  - If viewing a teammate: `killAllAgentsAndNotify()` + exit
    teammate view.
  - Otherwise if there's work to cancel: `handleCancel()`.
  - **Not claimed at the idle prompt** so the copy-selection
    handler and double-press-to-exit can see the keypress.

- **`chat:killAgents` (ctrl+x ctrl+k)** → `handleKillAgents`:
  - Two-press pattern with a 3000 ms confirmation window
    (`KILL_AGENTS_CONFIRM_WINDOW_MS`). First press shows a
    notification; second press within window clears the queue and
    kills running `local_agent` tasks via
    `killAllRunningAgentTasks`, emits `task_terminated` SDK events,
    and enqueues a single aggregate notification for the model.
  - **Must stay always-active** — ctrl+x is a chord prefix and
    cannot be deregistered without leaking ctrl+k to readline.

REPL's `onCancel` (`REPL.tsx:2106-2163`) is the business logic:

- Ignores elicitation dialogs (they own their Esc).
- Pauses proactive/KAIROS ticks.
- `queryGuard.forceEnd()` — bumps generation so the stale finally
  from the aborted `onQuery` can't reset state.
- Preserves partially-streamed text as an assistant message before
  `resetLoadingState` wipes `streamingText` (final order in
  transcript: [user, partial-assistant, "Request interrupted by
  user"]).
- Clears token budget snapshot.
- Dispatches the abort with reason-specific semantics:
  - `focusedInputDialog === 'tool-permission'` → `onAbort` on head
    of queue; clear queue.
  - `focusedInputDialog === 'prompt'` → reject all pending prompts,
    clear queue, abort controller with `'user-cancel'`.
  - `activeRemote.isRemoteMode` → `activeRemote.cancelRequest()`.
  - Otherwise → `abortController.abort('user-cancel')`.
- `setAbortController(null)` so subsequent Esc presses don't see a
  stale aborted signal.
- Fires `mrOnTurnComplete(…, true)` directly since forceEnd skipped
  the finally path.

Other abort-reason producers:
- `'background'` — `handleBackgroundSession` (`REPL.tsx:2528`) when
  the user backgrounds a running turn into a tmux pane.
- `'interrupt'` — `handlePromptSubmit`'s interruptible-tool path,
  the `'now'`-priority queued-command effect, or remote-session
  injected interrupts.

---

## 6. Other integration points

### 6.1 Remote / bridge / SSH

The REPL can be a *local* session, a *remote* session (WS to CCR
backend), a *direct-connect* session (WS to a claude server), or an
*SSH* session (child-process stdin/stdout to `claude ssh`). The three
hooks (`useRemoteSession`, `useDirectConnect`, `useSSHSession`,
`REPL.tsx:1389-1419`) have the same callback shape and are selected
via `activeRemote = sshRemote.isRemoteMode ? … : directConnect …
: remoteSession` (`1422`). `onSubmit` short-circuits to
`activeRemote.sendMessage` for plain text / image content; local-jsx
slash commands fall through for local rendering.

Separately, `useReplBridge` (`3833-3836`) mirrors the REPL's messages
to the bridge session for mobile clients (`claude.ai`). Its
`sendBridgeResult` callback is held in a ref so `onQuery`'s finally
can notify mobile clients that the turn ended without adding yet
another dep.

### 6.2 Scheduled / loop / teammate prompts

- `useInboxPoller` — swarm teammate messages delivered via mailbox;
  calls `handleIncomingPrompt` when idle.
- `useMailboxBridge` — analogous for non-swarm teammate mailboxes.
- `useScheduledTasks` — AGENT_TRIGGERS build; drives cron prompts.
- `useProactive` — KAIROS/PROACTIVE ticks; supports both
  "submit" (`onSubmitTick`) and "enqueue" (`onQueueTick`) modes.
- `useTaskListWatcher` — ant-only tasks mode.

All of these defer to queued user commands and the query guard so a
system prompt never clobbers a human's in-flight submission.

### 6.3 Session backgrounding / foregrounding

`useSessionBackgrounding` (constructed around
`REPL.tsx:2560+`) toggles `isExternalLoading` for the foregrounded
phase because the backgrounded turn resumes without a local
`queryGuard.reserve()` — the spinner and timing state need to
re-anchor without going through `onQuery`. `setIsExternalLoading`'s
wrapper exists for exactly this.

---

## 7. Ink-framework patterns that need explaining

**Yes, Ink usage in this file is non-obvious enough to warrant a
dedicated reviewer primer.** Specifics:

- **`useSyncExternalStore` for module-level stores.** React contexts
  under Ink can suffer propagation delays and dropped updates in
  rapid streams; `useSyncExternalStore` bypasses context and is used
  for the query guard (`REPL.tsx:904`), command queue
  (`useQueueProcessor.ts:43`), MCP clients, agents, and more. Each
  requires a matched `subscribe` / `getSnapshot` pair on the source
  module.
- **Ref mirrors for every "changes-frequently-but-used-only-in-
  callbacks" value.** See `streamModeRef` (`848`),
  `inputValueRef` (`1332`), `lastUserScrollTsRef` (`895`),
  `wasQueryActiveRef` (`949`), `focusedInputDialogRef` (`976`),
  `onSubmitRef` (`3613`), `restoreMessageSyncRef` (`877`),
  `sendBridgeResultRef` (`873`), `abortControllerRef` (`868`). Each
  has a comment explaining the dep-churn or re-render it avoids.
- **`useDeferredValue` for the transcript** (`1318`). Messages
  render at transition priority so the reconciler yields every 5 ms,
  keeping keystroke input responsive while a long `<Messages>` tree
  reconciles. `usesSyncMessages = showStreamingText || !isLoading`
  (`4506`) bypasses deferral when streaming or idle — deferral only
  pays off *during* streaming when there's work to lag behind.
- **`useTerminalFocus`, `useTabStatus`, `useTerminalTitle`,
  `useTerminalNotification`** (all from `src/ink/`). These wrap OS
  terminal APIs (ISTERM2 FOCUS report, OSC 21337 for sidebar/tab
  status, OSC 0 for titles, OSC 9 for bell). Non-standard; external
  maintainers won't recognize them from generic React/Ink samples.
- **`<AlternateScreen mouseTracking={…}>`** (`4485`, `5000`). Enters
  the xterm alt buffer and enables mouse tracking when fullscreen is
  on. The transcript-branch and main-branch both wrap in the same
  root type to let React reconcile across the toggle (otherwise the
  alt buffer would exit and re-enter, yanking the screen).
- **`<FullscreenLayout>`** with `scrollable` / `bottom` / `modal` /
  `overlay` / `bottomFloat` slots (`4565-4996`). Non-standard slot
  pattern — bottom is `flexShrink: 0`, scrollable uses an internal
  `ScrollBox`, overlay/modal sit absolute-positioned above. The
  sprite (`CompanionSprite`) flips between row-beside and column-
  stack layouts based on `transcriptCols < MIN_COLS_FOR_FULL_SPRITE`.
- **`useInput` is imported from `../ink.js`, not `ink` itself.** Same
  for `useStdin`, `useTheme`, `Box`, `Text` — the project ships a
  forked Ink with many custom hooks (`useTerminalFocus`,
  `useTabStatus`, …) and its own `render-border`, `stringWidth`,
  `hasCursorUpViewportYankBug`, etc. Reviewers expecting the public
  `ink` package will not find these APIs upstream.
- **`<KeybindingSetup>` as the outermost wrapper.** All keybindings
  are registered via `useKeybinding(name, handler, {context,
  isActive})` inside its context. Multiple contexts
  (`Chat`, `Global`, `Transcript`, …) are disjoint; handlers only
  fire when their context is active. `CancelRequestHandler` and
  `GlobalKeybindingHandlers` render *inside* `KeybindingSetup` so
  they can register.

Given the density of project-specific custom hooks and the ref/store
patterns layered on Ink, any reader not already intimate with this
codebase needs a dedicated "how Ink is used here" primer before they
can reason about the render cost of any change to `REPL.tsx`.

---

## 8. Performance-critical patterns that are easy to break

- **`setMessages` must update `messagesRef` synchronously**
  (`REPL.tsx:1198-1222`). Downstream logic (`handleSpeculationAccept
  → onQuery`, auto-restore rewind, file-history snapshots,
  `pickNewSpinnerTip`) reads `messagesRef.current` synchronously and
  would see stale data under the raw `useState` setter's batching.
- **`onSubmit`'s deps must NOT include `messages`.** `messages`
  changes ~30×/turn. `onSubmit` is prop-drilled into `PromptInput`,
  which caches; every recreation invalidates the whole input-
  component cache chain. Comment at `3538-3545` quantifies ~9 REPL
  scopes + ~15 messages arrays accumulating without this. Read
  messages through `messagesRef.current` instead.
- **`onSubmitRef`** (`3613-3621`) exists purely so
  `handleOpenRateLimitOptions` (prop-drilled to every MessageRow)
  stays stable. Without the ref, each MessageRow fiber pins an old
  REPL render scope at mount time — ~35 MB over 1000 turns.
- **Placeholder baseline** (`userInputBaselineRef`, `921-929`,
  `1198-1222`). The placeholder must hide the moment the real user
  message lands. Done by tracking a baseline length at submit and
  auto-incrementing baseline when async messages (bridge status,
  hook results) land during the gap, so only the human turn causes
  hide-out.
- **Token budget snapshotting on `onCancel`** (`2132-2136`). If the
  budget isn't cleared, the backstop can fire on a stale budget when
  the generator doesn't exit yet.
- **Clear abortController after cancel** (`2159`, `2993`). Otherwise
  `CancelRequestHandler`'s `canCancelRunningTask` reads `true` for
  the stale (aborted) signal, blocking Esc → double-press-exit flow.

---

## 9. Integration-point quick index

| File | Role |
|---|---|
| `src/screens/REPL.tsx` | Component, pipeline, cancel, auto-restore. |
| `src/components/PromptInput/PromptInput.tsx` | Input buffer UI, suggestions, typeahead, vim mode, image/text paste, directory completion, `@name` routing, speculation acceptance. |
| `src/components/TextInput.tsx` / `VimTextInput.tsx` | Raw buffer, cursor, keybindings. Emits `onSubmit(value)` on Enter. |
| `src/utils/handlePromptSubmit.ts` | `handlePromptSubmit` + `executeUserInput` — input validation, enqueue vs execute, `runWithWorkload`. |
| `src/utils/processUserInput/processUserInput.ts` | Mode dispatch: slash-command, bash, prompt, task-notification → message list. |
| `src/utils/QueryGuard.ts` | State-machine + subscribe API. |
| `src/utils/messageQueueManager.ts` | Module-level command queue + priority. |
| `src/utils/workloadContext.ts` | `runWithWorkload` AsyncLocalStorage. |
| `src/hooks/useCancelRequest.ts` | Keybinding registration. |
| `src/hooks/useQueueProcessor.ts` | Idle-drain queue processor. |
| `src/hooks/useReplBridge.ts` | Mobile-client bridge. |
| `src/hooks/useRemoteSession.ts` / `useDirectConnect.ts` / `useSSHSession.ts` | Remote transports. |
| `src/hooks/useSessionBackgrounding.ts` | Foreground/background of running turns. |
| `src/hooks/useMailboxBridge.ts`, `useInboxPoller.ts` | Teammate message injection. |
| `src/query.ts` | Async generator that produces the event stream `onQueryImpl` consumes. |
| `src/utils/messages.ts` | `handleMessageFromStream` converts events to messages / streaming state. |

---

## 10. Recommended follow-up deep dives

1. **`PromptInput` internals** — 2300 LOC covering typeahead,
   `useArrowKeyHistory`, `useHistorySearch`, `useTypeahead`,
   `useInputBuffer`, directory/command/slack/budget trigger
   detection, speculation acceptance, fast-mode picker, bridge
   dialog, model picker, background-tasks dialog, stash notice,
   footer/notifications/suggestions. Worth its own report.
2. **`processUserInput` dispatch tree** — how each mode (prompt /
   bash / background / task-notification / skill-expansion) becomes
   `{messages, shouldQuery, allowedTools, model, effort, nextInput}`.
   Includes image resizing, skill-frontmatter parsing, slash-command
   routing. The `isFirst` / `skipAttachments` semantics for batched
   turns should be covered there.
3. **`QueryGuard` invariants** — what transitions are legal, which
   calls are idempotent, how `generation` interacts with `end`,
   `forceEnd`, `cancelReservation`. Small module but load-bearing.
4. **Speculation + fast-mode acceptance path** — `handleSpeculationAccept`
   splices speculative messages into the real turn. Non-trivial
   interaction with `messagesRef` and `onQuery`'s append semantics.
5. **`query.ts` event stream** — a separate task already covers
   this, but the link into `onQueryEvent` /
   `handleMessageFromStream` is where the REPL pays the cost of
   every event shape.
6. **Keybinding context system** — `KeybindingSetup`,
   `useKeybinding`, `useKeybindings`, context activation rules.
   Dictates which handler wins when chords overlap (ctrl+x ctrl+k
   vs ctrl+x ctrl+e, Esc inside permission vs Esc at prompt, …).
7. **Ink fork diff** — cataloguing every custom hook and primitive
   in `src/ink/` (the project uses a forked Ink). Reviewers from
   outside the team will not recognize `useTabStatus`,
   `useTerminalFocus`, `useTerminalTitle`,
   `hasCursorUpViewportYankBug`, `AlternateScreen`, etc.

---

## 11. One-line summary

`REPL.tsx` is a single component that owns every session-wide piece
of UI and dispatches user keystrokes through
`PromptInput.onSubmit → REPL.onSubmit → handlePromptSubmit →
executeUserInput (runWithWorkload + processUserInput loop) → onQuery
→ onQueryImpl → query()` while keeping one `QueryGuard` state
machine as the single source of truth for "is the REPL busy" and
pervasively using ref-mirrors + `useSyncExternalStore` to prevent
Ink's render cost from compounding across a ~5000-LOC render tree.
