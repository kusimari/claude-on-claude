# MCP Server Initialization and Protocol Setup

**Task**: Wave 1 / Phase 1 / Task #2 — `mcp-bootstrap`
**Analyst**: analyst-4
**Scope**: Model Context Protocol (MCP) discovery, configuration merging, connection establishment, transport layer, authentication, lifecycle management, and tool integration.
**Primary files**: `src/services/mcp/` (23 files, ~12.3K LoC), `src/main.tsx` (launch wiring).

---

## 1. Executive Summary

MCP is Claude Code's plugin mechanism for integrating third-party tools, commands, resources, skills, prompts, and elicitations (user-prompt) into the agent loop. A given session may host **0–100+ MCP servers**, each communicating over one of seven transports (stdio, sse, http, ws, sse-ide, ws-ide, sdk, claudeai-proxy). Tools from those servers are surfaced with a normalized `mcp__{server}__{tool}` namespacing convention and registered via the same `Tool` interface as built-in tools — they participate in permission checks, auto-classifier grading, and streaming progress on equal footing.

**Core data structures**:
- `ScopedMcpServerConfig` (`services/mcp/types.ts:163`) — a server config tagged with one of seven `ConfigScope`s (local, user, project, dynamic, enterprise, claudeai, managed).
- `MCPServerConnection` (`types.ts:221`) — a discriminated union (`connected` / `failed` / `needs-auth` / `pending` / `disabled`) stored in `AppState.mcp.clients`.
- `AppState.mcp` — aggregates clients, tools, commands, resources; updated in batches by `useManageMCPConnections`.

**Control flow**: config discovery → policy/dedup filtering → transport-specific connection → capability negotiation → tool/resource/command/skill fetch → AppState update. Reconnection and hot-reload notifications (`tools/list_changed` etc.) keep the registry current.

---

## 2. Configuration Discovery and Precedence

### 2.1 Config sources (`ConfigScope` enum, `types.ts:10-20`)

| Scope        | Storage                                               | Added via                                  |
|--------------|-------------------------------------------------------|--------------------------------------------|
| `enterprise` | Managed file (`getManagedFilePath()/managed-mcp.json`) | Admin policy (exclusive if present)        |
| `user`       | Global config `mcpServers`                            | `claude mcp add -s user`                   |
| `project`    | `<cwd>/.mcp.json` (parent-dir walk)                   | Committed-in-repo config                   |
| `local`      | Current-project config `mcpServers`                   | `claude mcp add -s local` (gitignored)     |
| `dynamic`    | Runtime in-memory                                     | `--mcp-config` CLI flag, plugins           |
| `claudeai`   | Fetched from claude.ai connectors API                 | Auth cookie + eligible account             |
| `managed`    | Plugin MCP servers (scope `dynamic` w/ `pluginSource`) | Installed plugin                           |

### 2.2 Precedence and merge

`getClaudeCodeMcpConfigs(dynamicServers, extraDedupTargets)` in `config.ts:1071` performs the core merge:

1. If **enterprise config exists**, use only enterprise (exclusive control; `config.ts:1083-1096`).
2. Respect `isRestrictedToPluginOnly('mcp')` — keeps only plugin servers.
3. Read user, project, local scopes via `getMcpConfigsByScope()` (`config.ts:888`). Project scope walks from `cwd` upward through parent directories (`config.ts:914-955`); closer-to-cwd wins.
4. Load plugin MCP servers from `loadAllPluginsCacheOnly()`; errors collected into `PluginError[]`.
5. Filter project-scope servers to only `approved` (user-gated; `config.ts:1164-1170`).
6. Dedup plugin servers by signature against manual servers and earlier plugins (`dedupPluginMcpServers`, `config.ts:223-266`).
7. Final merge order (later overrides): `plugin < user < project < local`.
8. `filterMcpServersByPolicy` applied to all merged configs (`config.ts:536`) — `allowedMcpServers` / `deniedMcpServers` rules with name/command/URL matching.

`getAllMcpConfigs()` (`config.ts:1258`) adds claude.ai connectors fetched concurrently with `getClaudeCodeMcpConfigs()` via `fetchClaudeAIMcpConfigsIfEligible()` (`claudeai.ts`). Claude.ai connectors are deduped against manual by URL signature (`dedupClaudeAiMcpServers`, `config.ts:281-310`) — claude.ai has lowest precedence.

### 2.3 Signature-based dedup

`getMcpServerSignature(config)` (`config.ts:202-212`):
- stdio → `stdio:<JSON-stringified-command-array>`
- remote → `url:<unwrapped-vendor-URL>`

Remote signatures unwrap CCR proxy URLs (`unwrapCcrProxyUrl`, `config.ts:182`) so a plugin's raw vendor URL matches a connector's rewritten proxy URL pointing at the same server. This catches two configurations (e.g. manual `slack` plus claude.ai `Slack`) that would otherwise spawn duplicate connections + waste ~600 chars/turn in the system prompt.

### 2.4 Transport types (`types.ts:23-26`)

`'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk'` plus internal `'ws-ide'` and `'claudeai-proxy'`. Each has its own Zod schema (`types.ts:28-135`), validated on load and on `addMcpConfig()`. Environment variable expansion (`expandEnvVars`, `config.ts:556-616`) happens at parse time for stdio command/args/env and remote url/headers — missing vars become warnings but do not fail the load.

---

## 3. Connection Establishment (`client.ts:595`)

### 3.1 `connectToServer` (memoized)

Signature: `connectToServer(name, serverRef, serverStats?)` → `Promise<MCPServerConnection>` — cached by `getServerCacheKey(name, serverRef)` = `${name}-${JSON.stringify(serverRef)}`. Cache invalidated by `onclose` handler (`client.ts:1396`), `clearServerCache()` (`client.ts:1648`), or session expiry.

The function is a large dispatch on `serverRef.type`. For each transport it:

1. Builds a `Transport` object from the MCP SDK.
2. Wraps fetch with proxy + timeout + step-up detection for remote transports.
3. Creates an MCP `Client` with Claude Code identity + `roots` and `elicitation` capabilities (`client.ts:985-1002`).
4. Registers a `ListRootsRequestSchema` handler that returns `file://${getOriginalCwd()}` (`client.ts:1009-1018`).
5. Races `client.connect(transport)` against a `getConnectionTimeoutMs()` (default 30s, override via `MCP_TIMEOUT`) timeout.
6. On success: extracts capabilities, server version, truncates server instructions, registers handlers, returns `ConnectedMCPServer` with a `cleanup()` closure.
7. On `UnauthorizedError`: returns `needs-auth` via `handleRemoteAuthFailure()` (caches `needs-auth` state with 15-min TTL; `client.ts:340-361`).
8. On other failures: logs, closes transport, rethrows (caller converts to `failed`).

### 3.2 Transport-specific setup

**stdio** (`client.ts:944-958`): `StdioClientTransport` with `subprocessEnv()` + user-supplied `env`. `stderr: 'pipe'` so subprocess errors don't clobber the UI. Two hardcoded server names short-circuit to in-process: `claude-in-chrome` (`client.ts:905-924`) and Computer Use (`client.ts:925-943`) — both use `createLinkedTransportPair()` from `InProcessTransport.ts` to avoid a ~325MB subprocess spawn.

**sse** (`client.ts:619-677`): `SSEClientTransport` with `ClaudeAuthProvider` and two fetch wrappers. CRITICAL: `eventSourceInit.fetch` intentionally does NOT use the `wrapFetchWithTimeout` — the SSE stream is long-lived (60s timeout would kill it); timeout is only for POST/auth-refresh (`client.ts:643-647`).

**http** (`client.ts:784-865`): `StreamableHTTPClientTransport` with proxy dispatcher, step-up detection wrapping the innermost fetch, and session ingress token injected only when no OAuth tokens (`hasOAuthTokens` check, `client.ts:812`).

**ws** (`client.ts:735-783`) and **ws-ide** (`client.ts:708-734`): custom `WebSocketTransport` around Bun's native WebSocket (when available, with `proxy`/`tls`/`headers` options) or `ws` package fallback.

**sse-ide** (`client.ts:678-707`): Lightweight — IDE extensions run locally, no authentication (TODO: use lockfile auth token).

**claudeai-proxy** (`client.ts:868-904`): Routes through `${MCP_PROXY_URL}${MCP_PROXY_PATH}` with bearer token from `getClaudeAIOAuthTokens()`. Uses `createClaudeAiProxyFetch` (`client.ts:372-422`) which adds single-retry 401 handling via `handleOAuth401Error` — critical for preventing mass-401 under concurrent claude.ai connector connects.

**sdk** (`client.ts:866-867`): Raises an error; SDK servers are only instantiated through `setupSdkMcpClients` (see §7).

### 3.3 Connection timeout and drop detection

After connect, `client.onerror` and `client.onclose` are instrumented (`client.ts:1221-1402`):
- Terminal errors (`ECONNRESET`, `ETIMEDOUT`, `EPIPE`, `EHOSTUNREACH`, `ECONNREFUSED`, `Body Timeout Error`, SSE `Maximum reconnection attempts`) — counter; after `MAX_ERRORS_BEFORE_RECONNECT=3` forces close to trigger reconnect chain.
- `isMcpSessionExpiredError` (HTTP 404 + JSON-RPC `-32001`) fires immediate close + cache clear so next call reconnects with fresh session ID (`client.ts:193-215`, 1316-1329).
- `onclose` clears the connection cache entry, plus fetch-result caches for tools/resources/commands/skills (`client.ts:1383-1402`).

### 3.4 Cleanup

Each `ConnectedMCPServer.cleanup` is transport-specific (`client.ts:1404-...`):
- **In-process** (Chrome/Computer Use): `inProcessServer.close()` + `client.close()`.
- **stdio**: Removes stderr listener; sends SIGINT to child PID; polls for exit every 50ms; 600ms failsafe (`client.ts:1426-1481`).
- **Other transports**: `client.close()` cascades to `transport.close()` which runs the transport's own cleanup.

---

## 4. Capability Discovery & Tool Registration

### 4.1 Post-connect fetch (`getMcpToolsCommandsAndResources`, `client.ts:2226-2403`)

Orchestrates the connect-then-fetch batch:
1. Partitions configs into disabled vs active; disabled surfaces as `{type:'disabled'}` immediately.
2. Filters out servers with cached `needs-auth` (`isMcpAuthCached`, 15-min TTL) or `hasMcpDiscoveryButNoToken` — prevents re-probing under failed-auth state each session.
3. Splits active configs into local (stdio/sdk) vs remote, so each uses its own `processBatched` concurrency (different batch sizes — process spawning is heavier than network connects).
4. Per server, calls `connectToServer`; on `connected` returns, runs `Promise.all` for `fetchToolsForClient`, `fetchCommandsForClient`, optionally `fetchMcpSkillsForClient` (feature-gated `MCP_SKILLS`), and `fetchResourcesForClient` (if capability declared).
5. Emits `ListMcpResourcesTool` and `ReadMcpResourceTool` once (first server with resources).
6. Invokes `onConnectionAttempt` callback with `{client, tools, commands, resources}` so caller can update state.

### 4.2 Tool list → `Tool[]` (`fetchToolsForClient`, `client.ts:1743-1996`)

For each MCP tool (result of `client.request({method:'tools/list'}, ListToolsResultSchema)`):
- Construct fully qualified name `buildMcpToolName(serverName, toolName)` → `mcp__{normalized-server}__{normalized-tool}` (`mcpStringUtils.ts:50`; `normalization.ts:17`).
- If `skipPrefix` (SDK with `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`): use bare `tool.name` for model, keeping `mcpInfo` for permission routing.
- Read `_meta['anthropic/searchHint']` and `anthropic/alwaysLoad` for deferred-tool discovery.
- Map MCP hints to Tool methods: `readOnlyHint` → `isConcurrencySafe`/`isReadOnly`, `destructiveHint` → `isDestructive`, `openWorldHint` → `isOpenWorld`.
- `checkPermissions` returns a `passthrough` with a suggestion rule that writes `toolName: fullyQualifiedName` into `localSettings` on approval — this keeps per-server allowlists sticky across sessions.
- `call()` delegates to `callMCPToolWithUrlElicitationRetry` → `callMCPTool` (`client.ts:2813, 3029`). Wraps progress notifications, tool-use-ID injection, image+blob persistence, output truncation, and auth retry on 401.

Tool responses are sanitized (`recursivelySanitizeUnicode`, `client.ts:1758`) to prevent unicode-based injection and model confusion.

### 4.3 Commands, skills, resources

- **Commands** (`fetchCommandsForClient`, `client.ts:2033`) — MCP `prompts/list` surfaces as slash commands, with `argumentHint` derived from prompt arguments.
- **Skills** (`fetchMcpSkillsForClient` in `src/skills/mcpSkills.js`, feature-gated `MCP_SKILLS`) — discovered from `skill://` URI resources; integrated with local skill-search index.
- **Resources** (`fetchResourcesForClient`, `client.ts:2000`) — populates `ServerResource[]` under `AppState.mcp.resources[serverName]`.

All three fetchers are `memoizeWithLRU` with bound `MCP_FETCH_CACHE_SIZE = 20` and keyed by server name (stable across reconnects; caches cleared by `onclose`).

### 4.4 Hot-reload notifications (`useManageMCPConnections.ts:618-751`)

When a server declares `capabilities.tools.listChanged`, `prompts.listChanged`, or `resources.listChanged`, a notification handler is registered that:
1. Invalidates the relevant fetch cache.
2. Re-fetches concurrently (tools + skills + prompts as needed — skills are discovered from resources, so a resources/list_changed invalidates them too).
3. Calls `updateServer({...client, tools|commands|resources})` to push new data into AppState.
4. Logs `tengu_mcp_list_changed` analytics with before/after counts.

---

## 5. React Integration: `useManageMCPConnections` (`useManageMCPConnections.ts:143`)

Called via `<MCPConnectionManager>` context provider (`MCPConnectionManager.tsx:38`). Exposes `reconnectMcpServer` and `toggleMcpServer` via context hooks.

### 5.1 Lifecycle effects

**Effect 1: `initializeServersAsPending` (L772-854)** — Reads configs on session change or plugin reload. Uses `excludeStalePluginClients` to detect servers whose config hash changed or who vanished from config (e.g., removed plugin). Stale cleanup defuses three hazards before calling `clearServerCache`:
1. Pending reconnect timer must be cancelled (would fire with OLD config).
2. Existing `onclose` handler closes over OLD config; cleared first.
3. `clearServerCache` internally triggers `connectToServer` — for never-connected servers (disabled/pending/failed) this would spawn/OAuth just to kill it, so only connected servers get cleanup.

Stale servers get re-added as `pending` below; new configs seed as `pending` or `disabled`.

**Effect 2: `loadAndConnectMcpConfigs` (L858-1024)** — Two-phase connection:
- **Phase 1**: Kick off claude.ai fetch in parallel with `getClaudeCodeMcpConfigs`. Then call `getMcpToolsCommandsAndResources(onConnectionAttempt, enabledConfigs)` on regular configs.
- **Phase 2**: Await claude.ai, dedup against manual, add connectors to AppState as `pending`, connect them separately.

Phase 2 is skipped under `isStrictMcpConfig` or enterprise mode. Both phases filter out disabled servers before calling the connect loop.

### 5.2 Batched state updates

`updateServer` queues into `pendingUpdatesRef` and schedules a `setTimeout(flushPendingUpdates, 16ms)` flush (`useManageMCPConnections.ts:297-308`). A single `setAppState` call applies all queued updates — prevents render thrash when many servers connect in a rush.

### 5.3 Reconnection with exponential backoff (`useManageMCPConnections.ts:371-464`)

Triggered from `client.onclose` for remote transports (stdio/sdk don't reconnect). Up to `MAX_RECONNECT_ATTEMPTS = 5`, backoff `min(1000 * 2^(attempt-1), 30000)` ms. Checks `isMcpServerDisabled(name)` before each retry to honor concurrent disable. Pending timer stored in `reconnectTimersRef` for cancellation on unmount, reconfig, or explicit reconnect.

### 5.4 Channel notifications (Kairos, feature-gated)

On connect, gated by `gateChannelServer()` (allowlist, marketplace, dev, auth), registers handlers for:
- `notifications/claude/channel` (`ChannelMessageNotificationSchema`) — enqueues channel messages via `messageQueueManager.enqueue()` with `priority:'next'` and `origin:{kind:'channel', server}`.
- `notifications/claude/channel/permission` — resolves pending permission requests via `channelPermCallbacksRef.current.resolve()`.

Surfaced in AppState.channelPermissionCallbacks for `interactiveHandler` to reach.

---

## 6. Authentication (§ `auth.ts`, 2465 LoC)

### 6.1 `ClaudeAuthProvider` (`auth.ts:1376`)

Implements `OAuthClientProvider` from the MCP SDK:
- `tokens()` — Async; runs XAA silent-exchange path when `isXaaEnabled() && oauth.xaa && no refreshToken && (no accessToken || expires < 5min)` (`auth.ts:1585-1615`). Proactive refresh when `expiresIn <= 300s` via `refreshAuthorization()` (`auth.ts:1650-1667`). Refresh concurrency deduped via `_refreshInProgress` promise.
- `clientInformation()` — Reads from `secureStorage.mcpOAuth[serverKey]`, falls back to pre-configured `oauth.clientId`. Returns undefined to trigger DCR.
- `clientMetadataUrl` — Returns `MCP_CLIENT_METADATA_URL` (CIMD/SEP-991) so AS can fetch client metadata instead of doing DCR.
- `state()`, PKCE code verifier/challenge, redirect URI handling via `buildRedirectUri()`.
- `markStepUpPending(scope)` — Omits `refresh_token` in next `tokens()` so SDK falls through to PKCE (RFC 6749 §6 forbids scope elevation via refresh).

### 6.2 Step-up detection (`wrapFetchWithStepUpDetection`, `auth.ts:1354`)

Wraps the transport's inner fetch. When a remote returns 403 with WWW-Authenticate `insufficient_scope`, sets `_pendingStepUpScope` on the provider and `auth()`'s next invocation will start an interactive OAuth flow rather than attempting a no-op refresh.

### 6.3 XAA (Cross-App Access, SEP-990)

`performCrossAppAccess()` (`xaa.ts`) + `acquireIdpIdToken()` (`xaaIdpLogin.ts`): runs a 4-request silent exchange via enterprise IdP to trade an `id_token` for an MCP access token. Per-server `oauth.xaa: true` opt-in; IdP details (`issuer`, `clientId`, `callbackPort`) come from `settings.xaaIdp`.

### 6.4 OAuth callback flow

`createServer()` from `http` (`auth.ts:30`), port from `findAvailablePort()` — spins up a local HTTP server to receive the redirect, extracts `code` via `parse(url)`, kills the listener. Sensitive params (`state`, `nonce`, `code_challenge`, `code_verifier`, `code`) are redacted in logs.

### 6.5 MCP auth cache (`client.ts:257-316`)

`$CLAUDE_HOME/mcp-needs-auth-cache.json` with 15-min TTL per server. Prevents spam-probing of known-unauthenticated servers. Invalidated on `clearMcpAuthCache()` (e.g., post-login), write-serialized through `writeChain` to prevent concurrent RMW races.

---

## 7. SDK MCP Servers (`client.ts:3262` + `SdkControlTransport.ts`)

### 7.1 Setup

`setupSdkMcpClients(sdkMcpConfigs, sendMcpMessage)` is called from `print.ts` (headless/SDK mode). For each SDK server:
1. Creates `SdkControlClientTransport(name, sendMcpMessage)` — a Transport impl that tunnels MCP JSONRPC messages through SDK control requests.
2. Constructs `Client` with `capabilities: {}` (no roots/elicitation for SDK).
3. Connects, fetches tools, returns `ConnectedMCPServer` with scope `'dynamic'`.

### 7.2 Transport bridging

**`SdkControlClientTransport` (CLI side)** (`SdkControlTransport.ts:60`): `send()` calls `sendMcpMessage(serverName, message)` → await response → invoke `onmessage`. Control messages flow through stdin/stdout between CLI and SDK processes; Query resolves pending-request promises by ID.

**`SdkControlServerTransport` (SDK side)** (`SdkControlTransport.ts:109`): Pass-through — `onmessage` is called by Query when a control request arrives; `send()` invokes a callback to emit the response back through control channel.

Message IDs are preserved end-to-end so multiple in-flight calls don't cross-wire. The system supports N concurrent SDK servers.

### 7.3 In-process servers (`InProcessTransport.ts`)

`createLinkedTransportPair()` returns two `Transport` instances whose `send(msg)` on one delivers to `onmessage` on the other via `queueMicrotask`. Used for Chrome MCP (`client.ts:905-924`) and Computer Use (`client.ts:925-943`) — reserved names that run entirely in-process to avoid subprocess spawn overhead.

---

## 8. Integration with Main Launch (`main.tsx`)

### 8.1 Launch order (critical path through MCP)

`main.tsx:1414-1522` — Parse `--mcp-config` flag (JSON or file), validate via `parseMcpConfig`/`parseMcpConfigFromFilePath`, apply policy filter, wrap with scope `'dynamic'`.

`main.tsx:1784-1797` — Kick off `claudeaiConfigPromise` (for -p mode only; interactive uses the hook). `filterMcpServersByPolicy` on result.

`main.tsx:1803-1814` — `mcpConfigPromise = getClaudeCodeMcpConfigs(dynamicMcpConfig)` — overlaps with `setup()`, trust dialog, commands/agents loading.

`main.tsx:2380-2402` — After setup, await `mcpConfigPromise`. Split `sdkMcpConfigs` from `regularMcpConfigs`. Merge: `{...existingMcpConfigs, ...dynamicMcpConfig}` — CLI flag wins.

`main.tsx:2408-2430` — In interactive mode, call `prefetchAllMcpResources(regularMcpConfigs)` and parallel `prefetchAllMcpResources(claudeaiConfigs)`. Results merged, `uniqBy('name')` dedup helper tools. **Prefetch never blocks REPL render or turn-1 TTFT** — `mcpPromise.catch(() => {})` suppresses unhandled rejection; servers populate AppState async via `useManageMCPConnections` (same memoized `connectToServer` cache).

`main.tsx:2700-2720` (print mode path) — Direct call into `getMcpToolsCommandsAndResources` with a per-server-push callback into `headlessStore`, so ToolSearch can handle pending clients incrementally.

### 8.2 AppState integration

`AppState.mcp = { clients, tools, commands, resources, normalizedNames?, pluginReconnectKey }`. Tools flow into the global tool pool that feeds the API request (see Tool #14 analysis). Commands flow into the slash-command registry. Resources are fetched on demand by `ListMcpResourcesTool` / `ReadMcpResourceTool`.

---

## 9. Integration Points

| Subsystem | How MCP touches it |
|-----------|---------------------|
| **Tool system** (#14) | MCP tools are regular `Tool` instances with `isMcp: true`, `mcpInfo`, and `checkPermissions` returning passthrough with sticky localSettings rule suggestion |
| **Permission system** | `mcp__{server}__{tool}` qualified names; auto-mode classifier uses `mcpToolInputToAutoClassifierInput`; denied servers never spawn |
| **Plugin system** | Plugins inject MCP servers via `getPluginMcpServers`; deduped by signature; scoped `'dynamic'` with `pluginSource` |
| **Secure storage** | OAuth tokens, client info per `serverKey` in keychain/fallback; keychain cache hit-rate critical (tokens() called 30-40x/sec) |
| **Session ingress** | Remote sessions inject `sessionIngressToken` for URL-rewritten MCP proxy path through CCR |
| **Analytics** | Events: `tengu_mcp_servers`, `tengu_mcp_server_needs_auth`, `tengu_mcp_tools_commands_loaded`, `tengu_mcp_channel_message`, `tengu_mcp_list_changed`, `tengu_mcp_oauth_*`, `tengu_mcp_elicitation_shown`, etc. |
| **Message queue** | Channel notifications enqueue user messages as `priority:'next'`; participate in normal turn loop |
| **Hooks** (session/elicitation) | `executeElicitationHooks`, `executeElicitationResultHooks`, `executeNotificationHooks` — programmatic elicitation responses |
| **Cleanup registry** | `registerCleanup(client.ts)` ensures MCP cleanup runs on process exit |

---

## 10. Complexity Hotspots

1. **`client.ts:595-1647`** — The single `connectToServer` function exceeds 1000 LoC with per-transport branches. Consider refactor into per-transport factories (`stdio.ts`, `http.ts`, etc.) — see TODO on L589.
2. **`useManageMCPConnections.ts:858-1024`** — Two-phase load + stale cleanup + dedup + batched state update — state invariants tightly coupled to ordering; breakage risk high.
3. **`auth.ts`** (2465 LoC) — OAuth provider with PKCE, DCR, CIMD, XAA, step-up, 401-retry, proactive refresh, Slack 200-error normalization. Dense comments indicate many hard-won bug fixes; tests must cover cross-process refresh contention.
4. **Config dedup precedence** — Manual vs plugin vs claude.ai dedup is implemented in three separate sites (`dedupPluginMcpServers`, `dedupClaudeAiMcpServers`, `filterMcpServersByPolicy`); invariant that "the server that will actually connect wins" is easy to break.
5. **Reconnection + stale cleanup races** — Three numbered hazards documented at `useManageMCPConnections.ts:792-812` — each is a real bug that was fixed; small changes risk regressions.

---

## 11. Follow-up Recommendations

### 11.1 Deep-dive candidates (worth new tasks)

- **MCP OAuth / auth flows** — `auth.ts` at 2465 LoC has enough depth (PKCE, DCR, CIMD, XAA, step-up, token rotation) to warrant a dedicated analysis. Interacts with `secureStorage`, `lockfile`, `xaaIdpLogin`.
- **MCP tool call lifecycle** — `callMCPToolWithUrlElicitationRetry` → `callMCPTool` → `processMCPResult` → `transformMCPResult` (`client.ts:2713-3247`) handles progress, elicitation retry, image/blob persistence, validation, truncation. This is a separate control-flow cluster worth its own report.
- **Channel notifications** (`channelNotification.ts`, `channelPermissions.ts`, `channelAllowlist.ts`) — Kairos-gated feature where MCP servers can push messages into the turn queue and intercept permissions. Worth mapping the gate logic and message injection paths.
- **Elicitation** (`elicitationHandler.ts` 313 LoC) — Form and URL modes; queued into AppState; hooks-injectable. Small enough to cover in a focused follow-up.
- **Plugin-MCP integration** (`utils/plugins/mcpPluginIntegration.ts`) — Referenced but not read here.

### 11.2 Open questions

- What happens when `.mcp.json` is modified mid-session? `initializeServersAsPending` re-runs on session change (`/clear`) and plugin reload, but file-watcher on `.mcp.json` is not evident — probably requires `/mcp reload`.
- How are SDK MCP tool calls reconciled with the main tool-use stream for interleaved concurrent calls? `SdkControlTransport.ts` comment suggests Query handles correlation; verify in query.ts analysis.
- Cross-process OAuth contention: `lockfile.ts` usage and keychain TTL balancing — what's the actual observed P99 when 30+ concurrent claude.ai connectors connect?
- Is there a size cap on merged `pluginMcpServers`? Plugin MCP servers are not deduped by host resources — N heavy stdio servers from one plugin could blow subprocess limits.

### 11.3 Complexity ratings

| Area                        | Rating     |
|-----------------------------|------------|
| Config discovery/merge      | Complex    |
| Transport setup             | Complex    |
| Authentication (OAuth/XAA)  | Very complex |
| Lifecycle / reconnection    | Complex    |
| Tool / command / resource registration | Medium |
| SDK / in-process transports | Medium     |
| Elicitation / channels      | Medium (feature-gated) |

---

## Appendix A — Key file roles

| File | LoC | Role |
|------|----:|------|
| `client.ts` | 3348 | connectToServer, fetch*ForClient, callMCPTool, SDK setup |
| `auth.ts` | 2465 | ClaudeAuthProvider, OAuth/PKCE/DCR/XAA/step-up |
| `config.ts` | 1578 | discovery, scope merging, policy, dedup |
| `useManageMCPConnections.ts` | 1141 | React hook: init, connect, reconnect, notifications |
| `utils.ts` | 575 | helpers: stale detection, command belongs-to-server |
| `xaa.ts` | 511 | Cross-app access token exchange |
| `xaaIdpLogin.ts` | 487 | XAA IdP discovery + id_token cache |
| `channelNotification.ts` | 316 | Channel message gate + schema |
| `elicitationHandler.ts` | 313 | Form/URL elicitation + hooks |
| `types.ts` | 258 | Zod schemas, MCPServerConnection union |
| `channelPermissions.ts` | 240 | Permission-reply relay for channels |
| `claudeai.ts` | 164 | claude.ai connector fetch |
| `headersHelper.ts` | 138 | Static + dynamic header merge (headersHelper binary) |
| `SdkControlTransport.ts` | 136 | CLI↔SDK JSONRPC control bridge |
| `mcpStringUtils.ts` | 106 | mcp__server__tool parsing and building |
| `vscodeSdkMcp.ts` | 112 | VS Code–specific SDK MCP integration |
| `channelAllowlist.ts` | 76 | --channels allowlist |
| `oauthPort.ts` | 78 | Free-port finder, redirect URI builder |
| `officialRegistry.ts` | 72 | Registry prefetch |
| `MCPConnectionManager.tsx` | 72 | React context provider wrapping the hook |
| `InProcessTransport.ts` | 63 | Linked transport pair for in-process servers |
| `normalization.ts` | 23 | Name normalization for MCP protocol pattern |
| `envExpansion.ts` | 38 | Env-var expansion in config strings |

## Appendix B — MCP request timeouts

- `getConnectionTimeoutMs()` → `MCP_TIMEOUT` env var or 30000ms (`client.ts:456`)
- `MCP_REQUEST_TIMEOUT_MS = 60000` (auth, tool calls default)
- `AUTH_REQUEST_TIMEOUT_MS = 30000` (OAuth metadata discovery, refresh)
- `getMcpToolTimeoutMs()` → `MCP_TOOL_TIMEOUT` env or ~27.8h (`client.ts:224`)
- `MAX_MCP_DESCRIPTION_LENGTH` — truncates server instructions and tool descriptions
- `MCP_AUTH_CACHE_TTL_MS = 15*60*1000` — needs-auth skip window
- `MAX_RECONNECT_ATTEMPTS = 5`, `INITIAL_BACKOFF_MS = 1000`, `MAX_BACKOFF_MS = 30000` — reconnection
- `MCP_BATCH_FLUSH_MS = 16` — React state coalescing window
- `MCP_FETCH_CACHE_SIZE = 20` — LRU cap for tools/resources/commands/skills caches
