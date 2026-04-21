# 14 — Session Save/Restore, Conversation Persistence, and History

**Scope**: How Claude Code turns an in-memory conversation into a durable
artifact on disk, and how it rebuilds session state on `--continue`,
`--resume`, `--fork-session`, and CCR v2/ingress hydration. Also covers the
shell prompt-history file used by Up-arrow and ctrl+r. Centered on
`src/utils/sessionStorage.ts` (5105 LOC), `src/utils/sessionStoragePortable.ts`
(793 LOC), `src/utils/sessionRestore.ts` (551 LOC),
`src/utils/conversationRecovery.ts` (597 LOC), and `src/history.ts` (464 LOC).

---

## 1. On-disk shape

### 1.1 Directory layout

```
~/.claude/                                       ← getClaudeConfigHomeDir()
├── history.jsonl                                ← prompt history (shell-level)
└── projects/                                    ← getProjectsDir()
    └── -Users-foo-my-project/                   ← sanitized cwd, 0o700
        ├── <sessionId>.jsonl                    ← 0o600
        └── <sessionId>/
            └── subagents/
                └── [<subdir>/]agent-<agentId>.jsonl
```

- `getProjectsDir()` — `sessionStorage.ts:198-200`,
  `sessionStoragePortable.ts:325-327`.
- `getProjectDir(projectDir)` — memoized,
  `sessionStorage.ts:436-438`. `sessionStoragePortable.ts:329-331`
  duplicates the non-memoized version for VS Code / SDK which run under Node.
- `sanitizePath()` — `sessionStoragePortable.ts:311-319` replaces every
  non-alphanumeric char with `-`, then truncates + hash-suffixes if the
  component exceeds `MAX_SANITIZED_LENGTH=200` (fs component limit is
  255 bytes, leaves room for separator + hash). Bun uses `Bun.hash` /
  Node uses `djb2` — so long-path hash suffixes differ across runtimes;
  `findProjectDir()` (`sessionStoragePortable.ts:354-380`) falls back to
  prefix-matching for that exact reason.
- `canonicalizePath(dir)` — realpath + NFC normalize, so symlinked
  `/tmp` ↔ `/private/tmp` on macOS resolve to the same project dir
  (`sessionStoragePortable.ts:339-345`).

### 1.2 JSONL transcript format

Each session file is newline-delimited JSON. Every line is an `Entry`
discriminated by `type`. The exhaustive dispatch is in
`Project.appendEntry` (`sessionStorage.ts:1128-1265`):

| Entry `type`                     | Key fields                                                                                 | Written from                                       |
|----------------------------------|--------------------------------------------------------------------------------------------|----------------------------------------------------|
| `user`/`assistant`/`attachment`/`system` (TranscriptMessage) | `uuid`, `parentUuid`, `sessionId`, `cwd`, `version`, `gitBranch`, `userType`, `entrypoint`, `isSidechain`, `promptId`, `slug`, `agentId`, `teamName`/`agentName` | `insertMessageChain`                               |
| `summary`                        | `leafUuid`, `summary`                                                                       | AI-generated conversation summary                  |
| `custom-title`                   | `customTitle`, `sessionId`                                                                  | `/rename`, SDK `renameSession`, `saveCustomTitle`  |
| `ai-title`                       | `aiTitle`, `sessionId`                                                                      | `saveAiGeneratedTitle` — never re-appended         |
| `last-prompt`                    | `lastPrompt`, `sessionId`                                                                   | `reAppendSessionMetadata`                          |
| `task-summary`                   | `summary`, `sessionId`, `timestamp`                                                         | `claude ps` rolling snapshot                       |
| `tag`                            | `tag`, `sessionId`                                                                          | `/tag`, SDK `tagSession`                           |
| `agent-name` / `agent-color`     | `agentName` / `agentColor`, `sessionId`                                                     | `saveAgentName`, `saveAgentColor`                  |
| `agent-setting`                  | `agentSetting`, `sessionId`                                                                 | Cached at startup, written by `materializeSessionFile` |
| `mode`                           | `mode`, `sessionId`                                                                         | Coordinator/normal mode persistence                |
| `worktree-state`                 | `worktreeSession` (or null), `sessionId`                                                    | Worktree enter/exit                                |
| `pr-link`                        | `prNumber`, `prUrl`, `prRepository`, `sessionId`                                            | `linkSessionToPR`                                  |
| `file-history-snapshot`          | `messageId`, `snapshot`, `isSnapshotUpdate`                                                 | File edits + snapshot updates                      |
| `attribution-snapshot`           | `messageId`, cumulative state                                                               | `recordAttributionSnapshot` (ant-only)             |
| `content-replacement`            | `sessionId`, optional `agentId`, `replacements[]`                                           | Tool-result replacement records                    |
| `queue-operation`                | queue ops                                                                                   | `recordQueueOperation`                             |
| `speculation-accept`             | speculation data                                                                            | Speculative execution paths                        |
| `marble-origami-commit`          | `collapseId`, `summaryUuid`, `summaryContent`, `firstArchivedUuid`, `lastArchivedUuid`      | Context-collapse commit log (ordered)              |
| `marble-origami-snapshot`        | `staged`, `armed`, `lastSpawnTokens`                                                        | Context-collapse staged queue (last-wins)          |

File permissions: **files 0o600, dirs 0o700**, set at every append / mkdir
path (`appendToFile`, `appendEntryToFile`, `hydrateRemoteSession`,
`hydrateFromCCRv2InternalEvents`). `getFsImplementation()` lets the test
harness swap in a mock fs.

### 1.3 parentUuid chain invariants

The single source of truth is `isTranscriptMessage()`
(`sessionStorage.ts:139-146`) — only `user | assistant | attachment |
system` carry parentUuid. `isChainParticipant()`
(`sessionStorage.ts:154-156`) excludes `progress` entries from the chain
(progress is ephemeral UI state — see #14373, #23537 for what happened
when it *was* in the chain). Legacy transcripts still have progress
entries on disk; `loadTranscriptFile` bridges those via `progressBridge`
(`sessionStorage.ts:3623-3645`).

`insertMessageChain` (`sessionStorage.ts:993-1083`) walks the passed
array and stamps each message before append:

1. `parentUuid = startingParentUuid ?? null`.
2. For each message:
   - If it's a `tool_result` user message with `sourceToolAssistantUUID`,
     that UUID replaces the sequential parent. This is the **parallel-
     tool-result pattern** — multiple tool_result children share the
     same assistant message ancestor.
   - If the message is a compact-boundary, `parentUuid` is set to `null`
     and the original parent is kept under `logicalParentUuid` — this is
     what truncates the `--continue` chain at the boundary.
   - Session-stamp fields (`sessionId`, `cwd`, `userType`, `entrypoint`,
     `version`, `gitBranch`, `slug`) are written *after* the spread so
     forked messages don't retain their source session's stamps
     (comment `setup.ts:1049-1056`: without this, FRESH.jsonl ends up
     with mixed sessionIds and content-replacement lookups miss →
     FROZEN misclassification).
   - `await appendEntry(transcriptMessage)`.
   - If `isChainParticipant(message)`, advance `parentUuid = message.uuid`.

---

## 2. The Project singleton and the write pipeline

`Project` (`sessionStorage.ts:532-1384`) is a module-level singleton
created lazily by `getProject()` (`sessionStorage.ts:443-467`). On first
creation it registers one cleanup handler:

```ts
registerCleanup(async () => {
  await project?.flush()
  try { project?.reAppendSessionMetadata() } catch {}
})
```

### 2.1 Per-file batched write queue

```
appendEntry → enqueueWrite(filePath, entry)
                    ↓ push {entry, resolve}
         writeQueues: Map<filePath, Array<{entry, resolve}>>
                    ↓
     scheduleDrain() → setTimeout(…, FLUSH_INTERVAL_MS)
                    ↓
              drainWriteQueue():
                for each (filePath, queue):
                   join jsonStringify(entry)+'\n'
                   if content ≥ MAX_CHUNK_BYTES (100MB): flush + reset
                   appendFile(filePath, content, {mode: 0o600})
                   resolve() every waiter
                    ↓
              if writeQueues still has work → scheduleDrain() again
```

- `FLUSH_INTERVAL_MS = 100` (`sessionStorage.ts:567`), dropped to
  `REMOTE_FLUSH_INTERVAL_MS = 10` when CCR v2 or v1 session-ingress is
  enabled (`sessionStorage.ts:1350, 1360`).
- `MAX_CHUNK_BYTES = 100MB` (`sessionStorage.ts:568`) — caps a single
  `fs.appendFile` call; exceeding batches are split and flushed
  sequentially.
- `appendToFile` (`sessionStorage.ts:634-643`) retries once after
  `mkdir -p` if the first append ENOENTs; doesn't discriminate on the
  error code because NFS-like filesystems return surprising codes.
- `enqueueWrite` returns a `Promise<void>` resolved after the line
  actually hits disk — callers who need confirmation can `await` it;
  most (`appendEntry`) use `void` and rely on `trackWrite`.

### 2.2 Back-pressure: `trackWrite` + `flush`

Non-queue operations (`removeMessageByUuid`, `insertMessageChain`,
`insertFileHistorySnapshot`, `insertAttributionSnapshot`,
`insertContentReplacement`, `insertQueueOperation`) wrap their async
body in `trackWrite` (`sessionStorage.ts:597-604`), which increments
`pendingWriteCount` on entry and decrements on exit. When count hits
0, any `flush()` waiters are resolved.

`flush()` (`sessionStorage.ts:841-861`) is **ordered**:
1. Cancel the pending flush timer.
2. Await any in-flight drain.
3. Drain the queues once more.
4. If `pendingWriteCount > 0`, wait for the counter to hit 0 via
   `flushResolvers`.

This is the mechanism the exit cleanup handler relies on — `flush()`
completes only after every enqueue and every tracked op is persisted.

### 2.3 Buffered-entry path (pre-materialization)

`sessionFile` starts as `null`; entries enqueued before any user/
assistant message go into `pendingEntries: Entry[]`
(`sessionStorage.ts:552`). `appendEntry` short-circuits early in that
window (`sessionStorage.ts:1139-1142`) — the entry is buffered, not
written. `materializeSessionFile` (`sessionStorage.ts:976-991`) is
triggered from `insertMessageChain` the first time a user/assistant
message arrives, calls `ensureCurrentSessionFile()`, writes
`reAppendSessionMetadata()` (so `--name` / `--mode` / `--agent` survive
even a quit-before-message), then drains `pendingEntries` through
`appendEntry`. This is what prevents **metadata-only orphan files**.

### 2.4 Persistence gates (`shouldSkipPersistence`)

`sessionStorage.ts:960-970`:

- `NODE_ENV === 'test'` unless `TEST_ENABLE_SESSION_PERSISTENCE` is
  truthy.
- `getSettings_DEPRECATED()?.cleanupPeriodDays === 0`.
- `isSessionPersistenceDisabled()` (set by `--no-session-persistence`).
- `CLAUDE_CODE_SKIP_PROMPT_HISTORY` env var (set by tmuxSocket.ts so
  Tungsten-spawned tool verification sessions don't pollute `--resume`).

Guard is checked both in `appendEntry` and in `materializeSessionFile`
— `reAppendSessionMetadata` writes via `appendEntryToFile` (sync
`fs.appendFileSync`) which *bypasses* `appendEntry`, so without a
second guard a `--no-session-persistence` session would still create a
metadata-only file.

---

## 3. Metadata re-append: the 64KB tail invariant

The pickers (`readLiteMetadata`, VS Code `fetchSessions`, SDK) read only
the last 64KB of every session file to extract `customTitle`, `tag`,
`gitBranch`, PR link, and `lastPrompt`. `LITE_READ_BUF_SIZE = 65536`
(`sessionStoragePortable.ts:17`). If a session has been running long
enough that these fields have scrolled out of the tail, they become
invisible to the picker.

`reAppendSessionMetadata` (`sessionStorage.ts:721-839`) is the mechanism
that keeps them alive:

1. **Sync tail read** via `readFileTailSync` (same 64KB window).
2. **Refresh SDK-mutable fields from tail**: `customTitle` and `tag`
   can be mutated by an external SDK process (renameSession,
   tagSession). The CLI absorbs any fresher value from the tail into
   its own cache before re-appending — preventing its stale cached
   value from clobbering a concurrent rename. Filter is
   `startsWith('{"type":"custom-title"')` so a nested `"type":"tag"`
   inside a tool_use input isn't picked up.
3. **Unconditional re-append** of every cached field that's set. The
   ordering writes `last-prompt` first so the more-critical fields
   (title/tag/agent/mode/worktree) land closer to EOF.
4. Write via sync `appendEntryToFile` (`sessionStorage.ts:2572-2584`),
   not the queue — needs to finish before the process exits.

Called from three places:
- `materializeSessionFile` (`sessionStorage.ts:983`) — first user
  message.
- `adoptResumedSessionFile` (`sessionStorage.ts:1530-1534`) — on
  `--continue`/`--resume` with `skipTitleRefresh=true`.
- Exit cleanup (`sessionStorage.ts:458`) — via `registerCleanup`.
- Compaction paths (`compact.ts`, `reactiveCompact.ts` — out of scope)
  — writes metadata *before* the compact boundary so
  `scanPreBoundaryMetadata` can recover it on load.

Special cases:
- `ai-title` is intentionally **never re-appended**
  (`sessionStorage.ts:2640-2666`): a separate entry type so readers
  prefer `customTitle` over `aiTitle`, avoiding clobber-on-resume
  where an AI title would overwrite a user rename. Readers fall back
  to the head 64KB scan if the `ai-title` entry has been evicted.
- `worktree-state` is tri-state: `undefined = never touched`,
  `null = exited`, `object = currently in worktree`. Null means
  `--resume` won't cd back in.

---

## 4. Tombstoning (`removeMessageByUuid`)

`sessionStorage.ts:871-951` — used when streaming fails mid-message and
the partial entry is orphaned. **Fast path** (almost always taken):

1. Read last 64KB into a buffer.
2. `tail.lastIndexOf('"uuid":"${targetUuid}"')` — search for the full
   `"uuid":"…"` pattern, not bare UUID, so it can't match a
   `parentUuid` of a child. (UUIDs are pure ASCII so byte search is
   safe.)
3. Locate `prevNl` (LF at 0x0a — never appears inside UTF-8 multi-byte
   sequences, so byte-scan is correct across multi-byte chars).
4. `ftruncate(absLineStart)`; if there were trailing lines after the
   target, rewrite them positionally.

**Slow path**: target not in last 64KB. Reads the whole file, filters
out the matching UUID, writes back. Gated by
`MAX_TOMBSTONE_REWRITE_BYTES = 50MB` (`sessionStorage.ts:123`) — above
that, skip silently with a debug log. Session files can grow to
multiple GB (inc-3930), so this prevents OOM.

---

## 5. Load-side pipeline

### 5.1 Entry point: `loadTranscriptFile(path, opts)`

`sessionStorage.ts:3472-3813`. Returns maps keyed by sessionId/leafUuid
for every session-scoped value, plus `messages: Map<UUID, TranscriptMessage>`
and a pre-computed `leafUuids: Set<UUID>`.

Optional arg `opts.keepAllLeaves` is used by
`loadAllLogsFromSessionFile` (used by `/insights`) to build one
`LogOption` per leaf — otherwise the chain walker keeps only the most
recent leaf.

### 5.2 Pre-parse: 5MB threshold

`SKIP_PRECOMPACT_THRESHOLD = 5MB` (`sessionStoragePortable.ts:480`).
Files under it get a straight `readFile` + `parseJSONL`. Over it:

1. **`readTranscriptForLoad(filePath, fileSize)`**
   (`sessionStoragePortable.ts:717-793`) — a single forward chunked
   read (1MB chunks) that:
   - Strips `attribution-snapshot` lines at the fd level (never
     buffered); keeps only the most recent one, appended at EOF
     because `restoreAttributionStateFromSnapshots` reads only
     `[length-1]`.
   - Truncates the output buffer whenever a `compact_boundary`
     (without `preservedSegment`) is found — effectively "fast-forward
     to after the last boundary".
   - If the boundary *has* a `preservedSegment`, sets
     `hasPreservedSegment = true` and keeps buffering (the preserved
     messages are physically *pre*-boundary, so truncating would drop
     them).
   - Returns `{boundaryStartOffset, postBoundaryBuf, hasPreservedSegment}`.
   - Uses a straddle handler (`processStraddle`) for lines that cross
     a chunk boundary, and a growing `Sink` buffer that only doubles
     (capped at fileSize+1).
   - Motivation: a 151MB session that's 84% stale attr-snaps allocates
     ~32MB instead of 159+64MB. mimalloc doesn't return pages to the
     OS even after GC, so peak RSS matters.

2. **`scanPreBoundaryMetadata(filePath, endOffset)`**
   (`sessionStorage.ts:3157-3224`) — recovers session-scoped metadata
   that was physically before the boundary (`custom-title`, `tag`,
   `agent-*`, `mode`, `worktree-state`, `pr-link`, `summary`). Uses a
   byte-level forward scan: fast-path skip of any 1MB chunk that
   contains zero markers, then line-split within chunks that do have
   markers. Metadata entries are <50 per session, message content is
   99%+ — this keeps the overhead proportional to metadata, not
   message size.

### 5.3 `walkChainBeforeParse` — dead fork elimination

`sessionStorage.ts:3306-3466`. Every rewind/ctrl-z leaves an orphan
branch in the append-only log. `buildConversationChain` would discard
it later, but by then `parseJSONL` has paid to parse all of it.
Measured: 41MB, 99% dead → `parseJSONL` 56.0 ms → 3.9 ms (−93%).

Relies on two invariants verified across 25k+ message lines with zero
violations:
1. **Transcript messages always serialize with `parentUuid` as the
   first key** (insertion-order JSON.stringify + object literal).
   `{"parentUuid":` is a stable prefix that distinguishes transcript
   messages from metadata.
2. **Top-level uuid detection**: the uuid+timestamp suffix pattern
   `"uuid":"<36>","timestamp":"` is NOT unique — nested
   `agent_progress` data.message / server-controlled `mcpMeta` can
   both produce it. Disambiguated by `pickDepthOneUuidCandidate`
   (`sessionStorage.ts:3275-3304`), a string-aware brace-depth scan.

The walker builds a leaf→root chain, then only stitches the surviving
chain if it'd drop ≥50% of bytes (`len - chainBytes < len >> 1`).
Below break-even, concat memcpy dominates over parse savings (measured:
107MB session, 69% dead entries but only 30% dead bytes — not worth
it).

Gated off when:
- `opts.keepAllLeaves` is set (`/insights`).
- `hasPreservedSegment` is true (the splice is in-memory, post-parse —
  pre-parse walk would drop preserved messages as orphans).
- `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP` env var.
- Buffer is ≤5MB.

### 5.4 Post-parse: `applyPreservedSegmentRelinks` and `applySnipRemovals`

**`applyPreservedSegmentRelinks`** (`sessionStorage.ts:1839-1956`):
on compaction-with-preserved-segment, the preserved messages keep
their *pre-compact* parentUuids on disk (recordTranscript dedup-skipped
them — can't rewrite). In-memory splice:
- Find the absolute-last boundary and the last seg-boundary (can
  differ if a manual `/compact` followed a reactive compact).
- If the last seg is stale (no-seg boundary came after), skip relink
  but still prune at the absolute boundary.
- Validate the tail→head walk *before* mutating — malformed metadata
  becomes a no-op.
- Relink head's parent → anchor; relink anchor's *other* children →
  tail.
- **Zero stale usage**: assistant messages in the preserved range get
  their `input_tokens/output_tokens/cache_*` zeroed, because on-disk
  values reflect the ~190K pre-compact context and would otherwise
  trigger immediate auto-compact on resume.
- Prune everything physically before the absolute-last boundary that
  isn't in the preserved set.

**`applySnipRemovals`** (`sessionStorage.ts:1982-2039`): snip removes
middle ranges; JSONL is append-only so on-disk stays intact but
parentUuid chains walk through the gap. The boundary stores
`snipMetadata.removedUuids` for replay. Delete + path-compress
relink: for each survivor with a dangling parentUuid, walk backward
through `deletedParent` until hitting a non-removed ancestor. Without
this, `buildConversationChain` hits `messages.get(undefined)` and
orphans everything before the gap.

### 5.5 Leaf detection + chain build

`loadTranscriptFile` computes `leafUuids` two ways gated by GrowthBook
flag `tengu_pebble_leaf_prune`:
- Terminal = messages with no children (no parentUuid references them).
- For each terminal, walk back to the nearest user/assistant ancestor.
  Under `pebble_leaf_prune`, skip ancestors that already have a
  user/assistant child (prevents the "assistant tool_use with one
  progress child that's terminal, but a tool_result child that
  continues" case from falsely anchoring a resume).
- Cycle detection emits `tengu_transcript_parent_cycle`.

`buildConversationChain(messages, leafMessage)`
(`sessionStorage.ts:2069-2094`) walks `parentUuid` root-ward, detects
cycles, returns reversed. Post-pass `recoverOrphanedParallelToolResults`
inserts sibling tool_results / sibling assistant blocks that the
single-parent walk would miss.

### 5.6 Resume consistency beacon

`checkResumeConsistency(chain)` (`sessionStorage.ts:2224-2243`): scans
the reconstructed chain backward for the last `turn_duration` system
checkpoint, compares its `messageCount` to the chain's index at that
point, emits `tengu_resume_consistency_delta`. Positive delta = resume
loaded more than in-session (common failure); negative = truncation
(#22453 class).

---

## 6. Resume and continue flow

The entrypoint chain for CLI `--continue` / `--resume [id|path|title]`:

```
main.tsx action handler
    ↓
loadConversationForResume(source, sourceJsonlFile)   conversationRecovery.ts:456-597
    ├─ source=undefined ──→ loadMessageLogs()[0]   (skip live bg/daemon sessions via UDS)
    ├─ source=jsonl path ──→ loadMessagesFromJsonlPath(path)
    ├─ source=string ──→ getLastSessionLog(uuid)
    └─ source=LogOption ──→ (pre-loaded by picker)
    ↓
  isLiteLog(log) ? loadFullLog(log) : log
    ↓
  copyPlanForResume(log, sessionId)  // plan-files dir
  void copyFileHistoryForResume(log)
  messages = log.messages
  checkResumeConsistency(messages)
    ↓
  restoreSkillStateFromMessages(messages)
  deserializeMessagesWithInterruptDetection(messages):
     - migrateLegacyAttachmentTypes
     - strip invalid permissionMode
     - filterUnresolvedToolUses
     - filterOrphanedThinkingOnlyMessages
     - filterWhitespaceOnlyAssistantMessages
     - detect interrupted-prompt state
    ↓
  processSessionStartHooks('resume', {sessionId})
    ↓
processResumedConversation(result, opts, context)     sessionRestore.ts:409-551
    ├─ match coordinator mode (feature COORDINATOR_MODE)
    ├─ if !forkSession:
    │    switchSession(sid, transcriptDir)
    │    renameRecordingForSession()
    │    resetSessionFilePointer()
    │    restoreCostStateForSession(sid)
    ├─ else if contentReplacements:
    │    recordContentReplacement(contentReplacements)   // seed FRESH sessionId
    ├─ restoreSessionMetadata({...result})
    │    (strips worktreeSession on fork to avoid double-ownership)
    ├─ if !forkSession:
    │    restoreWorktreeForResume(result.worktreeSession)
    │    adoptResumedSessionFile()
    ├─ restoreFromEntries(contextCollapseCommits, contextCollapseSnapshot)  (CONTEXT_COLLAPSE)
    ├─ restoreAgentFromSession(agentSetting, ...)
    ├─ saveMode(coordinator|normal)
    ├─ computeRestoredAttributionState() (ant-only)
    ├─ computeStandaloneAgentContext(agentName, agentColor)
    └─ refreshAgentDefinitionsForModeSwitch(...)
    ↓
return ProcessedResume { messages, fileHistorySnapshots,
                         contentReplacements, agentName, agentColor,
                         restoredAgentDef, initialState }
```

### 6.1 `--continue` vs `--resume <id>` vs `--fork-session`

| Mode             | Source                                         | Session ID             | Worktree cd |
|------------------|------------------------------------------------|------------------------|-------------|
| `--continue`     | `loadMessageLogs()[0]`, skip live bg/daemon    | Reuse source sessionId | Yes         |
| `--resume <id>`  | `getLastSessionLog(id)`                        | Reuse source sessionId | Yes         |
| `--resume <path>`| `loadMessagesFromJsonlPath(path)`              | Reuse source sessionId | Yes         |
| `--resume <title>` (custom-title match) | `searchSessionsByCustomTitle(query)` | Reuse source sessionId | Yes |
| `--fork-session` | Same source, modified semantics                | Fresh startup sessionId | No (don't take ownership) |

Fork-specific quirks:
- Keeps the fresh startup sessionId — useLogMessages copies source
  messages into the new JSONL via `recordTranscript`.
- Content-replacement records are seeded via
  `recordContentReplacement(result.contentReplacements)` so the new
  sessionId's keyed lookup matches (otherwise fork → FROZEN
  misclassification).
- `worktreeSession` is stripped from `restoreSessionMetadata` — a
  "Remove" on fork exit would delete a worktree the original session
  still references.
- `adoptResumedSessionFile()` is *not* called — the lazy
  materialization path is correct (useLogMessages records to a new
  file).

### 6.2 `--session-id` interaction

`main.tsx:1279-1282`: `--session-id` can only combine with `--continue`
/ `--resume` if `--fork-session` is also specified. Otherwise it'd
collide with the source session's ID.

### 6.3 `--name` on resume

`restoreSessionMetadata` uses `??=` for `customTitle`
(`sessionStorage.ts:2773`) so a `cacheSessionTitle(--name)` call that
ran earlier wins over the stored title. REPL.tsx clears the cache
before calling for `/resume` so the slash-command version isn't
affected.

---

## 7. Remote and CCR v2 hydration

### 7.1 v1 Session Ingress

`persistToRemote` (`sessionStorage.ts:1302-1343`) — after every
transcript append, if `ENABLE_SESSION_PERSISTENCE=1` and a
`remoteIngressUrl` is set, POSTs the entry via
`sessionIngress.appendSessionLog(sessionId, entry, ingressUrl)`. On
failure emits `tengu_session_persistence_failed` and calls
`gracefulShutdownSync(1, 'other')` — this is a fatal error for
managed deployments.

`hydrateRemoteSession(sessionId, ingressUrl)`
(`sessionStorage.ts:1587-1622`): on resume from a remote environment,
fetches all entries, **truncates** the local file, writes remote logs.
Sets `remoteIngressUrl` in the finally block so persistence is enabled
only after sync is complete.

Agent-sidechain writes bypass remote (see line-1232 comment in
`appendEntry`): remote uses a single Last-Uuid chain per sessionId, so
re-POSTing a fork-inherited UUID 409s.

### 7.2 CCR v2 internal events

Alternative to Session Ingress. `setInternalEventWriter(writer)`
(`sessionStorage.ts:501-503`) registers a callback; when set,
transcript messages go through `internalEventWriter('transcript',
entry, {isCompaction, agentId})` instead of v1 Ingress. `FLUSH_INTERVAL_MS`
drops to 10ms.

`hydrateFromCCRv2InternalEvents(sessionId)`
(`sessionStorage.ts:1632-1723`):
1. `switchSession(sid)`.
2. Fetch foreground events via registered reader, write to session
   file (truncate).
3. If subagent reader exists, fetch + group by `agent_id`, write each
   agent's transcript to its own file.
4. Re-throw epoch mismatch (`CCRClient: Epoch mismatch (409)`) instead
   of swallowing — the worker must not race against graceful shutdown.

Server handles compaction filtering: it returns events starting from
the latest compaction boundary, so the client doesn't need to replay
the full log.

---

## 8. Listing sessions for the resume picker

### 8.1 Three-tier progressive enrichment

`fetchLogs` / `loadMessageLogs` / `loadSameRepoMessageLogsProgressive`
all follow the same pattern:

1. **`getSessionFilesLite(projectDir, limit)`** — pure stat, zero file
   reads. Returns `LogOption[]` with `isLite:true` and empty
   `messages:[]`. Instant even for thousands of sessions.
2. **`enrichLogs(allStatLogs, 0, initialEnrichCount)`** — reads head
   + tail (64KB each) of each lite log via `readLiteMetadata`. Extracts
   `firstPrompt`, `gitBranch`, `customTitle`, `tag`, PR link from JSON
   string-field scraping (no full parse, tolerates truncated lines).
   `INITIAL_ENRICH_COUNT = 50` — ~6.4MB I/O total for the initial
   picker view.
3. **`loadFullLog(liteLog)`** — called when the user actually selects
   a session. Runs `loadTranscriptFile` on the `fullPath`, builds the
   chain, populates everything.

`enrichLogs` returns `nextIndex` for progressive loading — the picker
UI fetches more as the user scrolls.

### 8.2 `readLiteMetadata` fallback chain

`sessionStorage.ts:4739-4813`. firstPrompt priority:
1. `extractLastJsonStringField(tail, 'lastPrompt')` — authoritative
   (filtered by `extractFirstPrompt` at write time).
2. `extractFirstPromptFromChunk(head)` — head scan with command and
   bash prefix handling, tick-message detection.
3. Raw `content` / `text` field scrapes (catches VS Code
   `<ide_selection>` metadata + array-format content blocks).
4. `'(session)'` placeholder so crashed sessions still appear.

customTitle priority:
1. `customTitle` field in tail (user rename — highest priority).
2. `customTitle` field in head.
3. `aiTitle` field in tail.
4. `aiTitle` field in head (ai-title never re-appended, falls out of
   tail over time).

### 8.3 Sidechain + teamName filtering

`enrichLog` (`sessionStorage.ts:5023-5070`) filters out `isSidechain`
and sessions with `teamName` set — subagent / teammate transcripts
shouldn't clutter `--resume`. `getSessionFilesWithMtime`
(`sessionStorage.ts:4526-4569`) skips non-UUID filenames.

### 8.4 Branching analytics

`trackSessionBranchingAnalytics` (`sessionStorage.ts:2526-2557`) counts
how many logs share the same sessionId — this is how sessions "fork"
in the log listing (same sessionId, different leafUuids) and fires
`tengu_session_forked_branches_fetched` when any branching exists.

### 8.5 Sibling worktree search

`resolveSessionFilePath(sessionId, dir?)`
(`sessionStoragePortable.ts:403-466`): if the session isn't in
`projectDir`, scans `getWorktreePathsPortable(canonical)` for sibling
worktrees. `searchSessionsByCustomTitle` / `loadSameRepoMessageLogs`
use this to make sessions created under any worktree of the same
repo findable from any other worktree.

---

## 9. Prompt history file (`src/history.ts`)

Separate system from session transcripts — this is the shell-style
command history for Up-arrow and ctrl+r.

### 9.1 Shape

`~/.claude/history.jsonl` (global, not per-project). Each line:

```ts
type LogEntry = {
  display: string
  pastedContents: Record<number, StoredPastedContent>
  timestamp: number      // Date.now()
  project: string        // getProjectRoot()
  sessionId?: string
}
```

`MAX_HISTORY_ITEMS = 100` (`history.ts:19`). Scan window; oldest
entries beyond 100 in the current-project list are never shown.

### 9.2 Pasted-content storage

`MAX_PASTED_CONTENT_LENGTH = 1024` (`history.ts:20`). Small pastes are
inlined in the history entry (`content` field). Large pastes compute
`hashPastedText(content)` synchronously, store the `contentHash`
reference, and fire-and-forget `storePastedText(hash, content)` to
`pasteStore`. Resolved lazily via `resolveStoredPastedContent`.

Images are never stored inline — they go through `image-cache` and
are filtered out here.

### 9.3 Write path

`addToHistory(command)` → `addToPromptHistory`:
1. Skip if `CLAUDE_CODE_SKIP_PROMPT_HISTORY` (Tungsten-spawned
   sessions).
2. Register cleanup on first use (await flush + final flush for
   anything pending).
3. Build `LogEntry`, push to `pendingEntries` array.
4. `flushPromptHistory(0)` fires.

`immediateFlushHistory` (`history.ts:292-327`):
1. `writeFile(historyPath, '', {flag: 'a'})` to ensure exists.
2. `lock(historyPath, {stale: 10s, retries: 3 × 50ms})` — proper
   filesystem lock (distinct from session JSONL which has no lock
   because one session has one file).
3. Serialize pending + append + release lock.

`flushPromptHistory(retries)` (`history.ts:329-353`) retries up to 5
times with 500ms sleeps; stops trying until next prompt if >5.

### 9.4 Read path

Three readers, all use `makeLogEntryReader` / `readLinesReverse`:
- `getHistory()` (Up-arrow) — current-project entries, current-session
  entries yielded first so concurrent sessions don't interleave.
- `getTimestampedHistory()` (ctrl+r) — current-project, deduped by
  display text, lazy paste resolution.
- `makeHistoryReader()` (generic) — all entries.

`pendingEntries` are read first (reverse order) so entries not yet
flushed are visible immediately.

### 9.5 `removeLastFromHistory`

Auto-restore-on-interrupt case: Esc rewinds before response arrives,
and the history entry should be undone too (otherwise Up-arrow shows
restored text twice).

Fast path: `pendingEntries.splice(idx, 1)`. Slow path (async flush
already won the TTFT race): add the entry's timestamp to
`skippedTimestamps` — consulted by `makeLogEntryReader` to filter on
read. One-shot via `lastAddedEntry = null`.

---

## 10. Key data structures summary

| Structure | Where | Notes |
|---|---|---|
| `Project` singleton | `sessionStorage.ts:440` | Module-level, lazy |
| `writeQueues: Map<filePath, {entry, resolve}[]>` | `Project.writeQueues` | Per-file batching |
| `pendingEntries: Entry[]` | `Project.pendingEntries` | Pre-materialization buffer |
| `sessionFile: string \| null` | `Project.sessionFile` | Lazily set on first msg |
| `currentSession{Tag,Title,AgentName,AgentColor,LastPrompt,AgentSetting,Mode,Worktree,Pr*}` | `Project.currentSession*` | Cache for re-append |
| `existingSessionFiles: Map<sessionId, path>` | `Project.existingSessionFiles` | Stat cache for other-session writes |
| `pendingEntries` (prompt history) | `history.ts:281` | Before flush |
| `skippedTimestamps: Set<number>` | `history.ts:289` | removeLastFromHistory slow path |
| `getSessionMessages: memoized Map<sessionId, Set<UUID>>` | `sessionStorage.ts:3842` | UUID dedup on append |

---

## 11. Integration points

- **Bootstrap state** (`bootstrap/state.ts`): `getSessionId`,
  `getSessionProjectDir`, `switchSession`, `getOriginalCwd`,
  `getPromptId`, `getPlanSlugCache`, `isSessionPersistenceDisabled`.
  Session ID + project dir are atomic (CC-34) to avoid the gh-30217
  bug where transcript_path was computed with originalCwd but the
  file was written to sessionProjectDir.
- **Cleanup registry** (`utils/cleanupRegistry.ts`): `registerCleanup`
  — exit handler flushes queue + re-appends metadata.
- **Graceful shutdown** (`utils/gracefulShutdown.ts`): `isShuttingDown()`
  blocks remote persistence during shutdown; `gracefulShutdownSync(1)`
  is the escalation when remote Ingress fails.
- **Worktree** (`utils/worktree.ts`): `worktree-state` entry persists
  the session's current worktree. `restoreWorktreeForResume` cd's
  back into it on resume.
- **Plans** (`utils/plans.ts`): `copyPlanForResume(log, sessionId)`
  duplicates per-session plan files when forking or resuming across
  sessions. `getPlanSlugCache` provides the slug stamped on every
  transcript message.
- **File history** (`utils/fileHistory.ts`): `file-history-snapshot`
  entries; `copyFileHistoryForResume` duplicates snapshots to the
  forked session.
- **Attribution** (`utils/commitAttribution.ts`): `attribution-snapshot`
  entries (ant-only); the fd-level skip in `readTranscriptForLoad`
  was designed for this — they're the biggest stale-byte contributor.
- **Context collapse / marble-origami**
  (`services/contextCollapse/persist.ts`): `marble-origami-commit` /
  `marble-origami-snapshot` entries. `restoreFromEntries(commits,
  snapshot)` rebuilds the collapsed-span view from the log.
- **Todo V1** (`utils/todo/*`): on SDK/non-interactive resume,
  `extractTodosFromTranscript` scans the last TodoWrite tool_use to
  rehydrate AppState.todos. V2 uses file-backed tasks instead.
- **Analytics** (`services/analytics/*`): `tengu_*` events —
  `tengu_session_persistence_failed`, `tengu_resume_consistency_delta`,
  `tengu_transcript_parent_cycle`, `tengu_chain_parent_cycle`,
  `tengu_session_renamed`, `tengu_session_tagged`,
  `tengu_session_linked_to_pr`, `tengu_agent_name_set`,
  `tengu_agent_color_set`, `tengu_snip_resume_filtered`,
  `tengu_relink_walk_broken`, `tengu_session_forked_branches_fetched`.
- **Hooks** (`utils/sessionStart.ts`): `processSessionStartHooks('resume',
  ...)` runs on resume; hook output is appended to the conversation.
- **Cost tracker** (`cost-tracker.ts`): `restoreCostStateForSession(sid)`
  on non-fork resume.
- **Session ingress** (`services/api/sessionIngress.ts`): v1 remote
  protocol; `appendSessionLog` / `getSessionLogs`.
- **Asciicast** (`utils/asciicast.ts`): `renameRecordingForSession()`
  after switchSession so `/share` finds the recording under the
  resumed sessionId.

---

## 12. Notable invariants and subtleties

- **Session ID + projectDir atomicity** (CC-34): switched together via
  `switchSession(sid, projectDir)`. Without this, sessions saved under
  one path became invisible to hooks that looked up
  `getTranscriptPathForSession(sid)` using the other.
- **parentUuid is first key in JSON serialization**: load-time dead-
  fork pruning relies on `{"parentUuid":` prefix. Break this and
  `walkChainBeforeParse` silently treats every line as metadata.
- **Compact boundary semantics**: `parentUuid = null` on the boundary
  and `logicalParentUuid = originalParent`. On load, boundary is how
  `--continue` knows where to truncate; `logicalParentUuid` is how
  /share can still show the pre-compact chain on explicit request.
- **Session-stamp re-stamping** on fork/resume: messages arrive as
  SerializedMessage with original `sessionId`, `cwd`, `version`, etc.
  `insertMessageChain` spreads the message first, then overrides —
  ordering matters. Mixed sessionId is the observed failure mode
  (FROZEN content-replacement misclassification).
- **Agent sidechain bypasses dedup**: main-file UUID set is
  authoritative; agent sidechain writes to a separate file and must
  not poison the set, or the main-thread message chains parent to a
  UUID that only exists in the sidechain file → --resume terminates
  at the dangling ref.
- **`ai-title` never re-appended**: distinct entry type, separate
  `aiTitle` field, readers prefer `customTitle`. Guarantees a user
  rename is never clobbered by a stale AI title on resume.
- **Zero usage on preserved segment messages**: on-disk
  input_tokens/output_tokens/cache_* reflect pre-compact context;
  without zeroing, resume → immediate auto-compact spiral.
- **`lock` on history.jsonl, no lock on transcripts**: transcripts
  are per-session so there's no writer contention; history is global
  and concurrent sessions share it.
- **64KB windowed reads everywhere that lists sessions**: head
  64KB + tail 64KB. Metadata that falls out of the tail is invisible
  unless re-appended. `reAppendSessionMetadata` is the invariant
  enforcer; `scanPreBoundaryMetadata` is the recovery mechanism for
  compaction-pushed content.
- **5MB precompact threshold**: below, `readFile` + `parseJSONL`.
  Above, forward chunked read + attr-snap skip + boundary truncate +
  pre-boundary metadata scan + optional dead-fork prune.
- **CCR v2 epoch mismatch re-throw**: the worker must observe it and
  call gracefulShutdown; swallowing would race.

---

## 13. Follow-up deep dives recommended

1. **Compaction (`compact.ts`, `reactiveCompact.ts`)** — the
   `compact_boundary` / `preservedSegment` write path is out of scope
   here but tightly coupled to loadTranscriptFile's boundary handling
   and reAppendSessionMetadata's pre-boundary metadata placement.
2. **`deserializeMessagesWithInterruptDetection`
   (`conversationRecovery.ts:164-…`)** — legacy attachment
   migrations, unresolved-tool-use filtering, thinking-only filter,
   interrupt detection. Overlaps with interrupts (analyst-5's task).
3. **Context-collapse persist
   (`services/contextCollapse/persist.ts`)** — the
   `marble-origami-*` entries and `restoreFromEntries` logic.
4. **Content-replacement storage
   (`utils/toolResultStorage.ts`)** — how tool-result replacements
   are computed, how FROZEN vs not-FROZEN is classified on resume.
5. **File history (`utils/fileHistory.ts`)** — snapshot creation,
   `isSnapshotUpdate` chain, `fileHistoryRestoreStateFromLog`.
6. **Remote Session Ingress protocol
   (`services/api/sessionIngress.ts`)** and CCR v2 internal-event
   client — the network layer that backs remote hydration.
7. **Resume picker UI
   (`screens/ResumeConversation.tsx`, `commands/resume/resume.tsx`)**
   — how progressive enrichment is consumed.
8. **Paste store (`utils/pasteStore.ts`)** — large paste hashing and
   retention.

---

## 14. Flagged: persistence strategy complexity

This deliverable's brief asked to flag "if persistence strategy is
complex." **It is.** Signal of that complexity, beyond the ~5000 LOC:

- 5 separate persistence paths (local JSONL, agent sidechains, v1
  Session Ingress, CCR v2 internal events, prompt history).
- Append-only log with two distinct in-memory filtering mechanisms
  (compaction's `preservedSegment` splice, snip's `removedUuids`
  relink) that have been iterated on in response to production
  incidents (#22453, adamr-20260320, inc-3930, inc-3694, inc-4718).
- File-size-gated load strategies (<5MB → readFile+parse,
  ≥5MB → forward chunked with fd-level skip, plus optional
  dead-fork pre-parse walk).
- Two JSON-field-scraping utilities
  (`extractJsonStringField`, `extractLastJsonStringField`) that work
  on truncated lines because tail reads can slice mid-line.
- Metadata re-append choreography centered on a 64KB tail window,
  with SDK/CLI collaboration via tail-refresh + external-writer
  absorption.
- Tombstone fast path (64KB tail search + truncate) and slow path
  (rewrite, bounded at 50MB) because session files legitimately
  grow to GB.
- Fork vs resume vs continue have distinct ownership semantics for
  worktree-state, contentReplacements, sessionId, and plan files.
- Legacy data migrations on read (progress bridge, attachment type
  rewrites, permissionMode validation, displayPath backfill).

Individually each is justified; collectively it's the most
carefully-engineered subsystem examined so far.

---

## 15. Code references quick index

| Concern                               | File:line                                    |
|---------------------------------------|----------------------------------------------|
| `Project` singleton                   | `utils/sessionStorage.ts:440-467, 532-1384` |
| Write queue drain                     | `utils/sessionStorage.ts:618-686`            |
| `appendEntry` type dispatch           | `utils/sessionStorage.ts:1128-1265`          |
| `insertMessageChain`                  | `utils/sessionStorage.ts:993-1083`           |
| `materializeSessionFile`              | `utils/sessionStorage.ts:976-991`            |
| `reAppendSessionMetadata`             | `utils/sessionStorage.ts:721-839`            |
| `removeMessageByUuid` (tombstone)     | `utils/sessionStorage.ts:871-951`            |
| `shouldSkipPersistence`               | `utils/sessionStorage.ts:960-970`            |
| `adoptResumedSessionFile`             | `utils/sessionStorage.ts:1530-1534`          |
| `hydrateRemoteSession`                | `utils/sessionStorage.ts:1587-1622`          |
| `hydrateFromCCRv2InternalEvents`      | `utils/sessionStorage.ts:1632-1723`          |
| `buildConversationChain`              | `utils/sessionStorage.ts:2069-2094`          |
| `applyPreservedSegmentRelinks`        | `utils/sessionStorage.ts:1839-1956`          |
| `applySnipRemovals`                   | `utils/sessionStorage.ts:1982-2039`          |
| `checkResumeConsistency`              | `utils/sessionStorage.ts:2224-2243`          |
| `loadTranscriptFile`                  | `utils/sessionStorage.ts:3472-3813`          |
| `walkChainBeforeParse`                | `utils/sessionStorage.ts:3306-3466`          |
| `scanPreBoundaryMetadata`             | `utils/sessionStorage.ts:3157-3224`          |
| `loadFullLog`                         | `utils/sessionStorage.ts:2949-3056`          |
| `loadAllLogsFromSessionFile`          | `utils/sessionStorage.ts:4598-4694`          |
| `readLiteMetadata`                    | `utils/sessionStorage.ts:4739-4813`          |
| `getSessionFilesLite` (stat-only)     | `utils/sessionStorage.ts:4975-5016`          |
| `enrichLogs` (progressive)            | `utils/sessionStorage.ts:5077-5105`          |
| `readTranscriptForLoad`               | `utils/sessionStoragePortable.ts:717-793`    |
| `sanitizePath` + long-path hash       | `utils/sessionStoragePortable.ts:311-319`    |
| `resolveSessionFilePath` + worktree   | `utils/sessionStoragePortable.ts:403-466`    |
| `LITE_READ_BUF_SIZE`                  | `utils/sessionStoragePortable.ts:17`         |
| `SKIP_PRECOMPACT_THRESHOLD` (5MB)     | `utils/sessionStoragePortable.ts:480`        |
| `loadConversationForResume`           | `utils/conversationRecovery.ts:456-597`      |
| `processResumedConversation`          | `utils/sessionRestore.ts:409-551`            |
| `restoreWorktreeForResume`            | `utils/sessionRestore.ts:332-366`            |
| `restoreAgentFromSession`             | `utils/sessionRestore.ts:200-242`            |
| Cleanup handler (flush + re-append)   | `utils/sessionStorage.ts:448-462`            |
| Prompt history `addToHistory`         | `history.ts:411-434`                         |
| `immediateFlushHistory`               | `history.ts:292-327`                         |
| `removeLastFromHistory`               | `history.ts:453-464`                         |
