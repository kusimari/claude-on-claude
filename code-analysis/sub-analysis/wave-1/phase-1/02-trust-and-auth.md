# Task 02 — Trust & Authentication Flows (`showSetupScreens` + auth stack)

**Scope:** the setup-dialog orchestrator `showSetupScreens()` and everything it gates on — workspace trust, onboarding, OAuth / API-key / third-party auth, custom-key approval, enterprise SSO, token storage and refresh. This is large enough that the downstream OAuth refresh+401 machinery likely deserves its own follow-up task (flagged at the end).

---

## 1. Executive summary

`showSetupScreens()` (src/interactiveHelpers.tsx:104–298) is the **interactive-mode gate** that every TTY session traverses before the REPL boots. Its architectural function is not to configure the app but to **establish a layered trust boundary**:

1. **Onboarding** collects theme + marks `hasCompletedOnboarding` (one-time).
2. **TrustDialog** is the workspace trust boundary — accepting it unlocks dangerous env vars, OAuth-dependent helpers, and GrowthBook auth headers.
3. **Post-trust dialogs** (MCP approvals, CLAUDE.md external includes, Grove policy, ApproveApiKey, BypassPermissionsModeDialog, AutoModeOptInDialog, DevChannelsDialog, ClaudeInChromeOnboarding) run in a fixed order so each can assume the prior gate has resolved.

Trust is **per-directory, persisted to disk for project paths and session-only for $HOME**. Authentication is separate from trust: OAuth tokens and API keys follow their own cascade (`src/utils/auth.ts`), can pre-exist the trust dialog (CLI flag, env var, secure storage), and are **used** only after trust fires because `apiKeyHelper`, `otelHeadersHelper`, and `applyConfigEnvironmentVariables()` are explicitly gated on it.

CI / `-p` print mode never reaches `showSetupScreens` at all (the interactive orchestrator is short-circuited in `main.tsx`), so this file is exclusively a TTY pipeline.

---

## 2. `showSetupScreens()` — orchestration pipeline

File: `src/interactiveHelpers.tsx:104–298`. Called from `src/main.tsx:2241` inside the interactive boot path.

Signature:
```ts
showSetupScreens(
  root: Root,
  permissionMode: PermissionMode,
  allowDangerouslySkipPermissions: boolean,
  commands?: Command[],
  claudeInChrome?: boolean,
  devChannels?: ChannelEntry[],
): Promise<boolean /* onboardingShown */>
```

### 2.1 Short-circuit at the top (lines 105–108)
Returns `false` immediately in three cases that must never render dialogs:
- build-time constant `"production" === 'test'` (dead in release builds),
- `isEnvTruthy(false)` (placeholder for a disabled env flag — literal `false` makes this dead; leftover from a removed kill-switch),
- `process.env.IS_DEMO` truthy (scripted demos).

### 2.2 Onboarding (lines 109–123)
Runs iff `!config.theme || !config.hasCompletedOnboarding`. Sets local `onboardingShown = true` (returned at the end — callers use it to alter later modal copy, e.g. Grove uses `location: onboardingShown ? 'onboarding' : 'policy_update_modal'`). Dialog `onDone` calls `completeOnboarding()` (lines 32–38) which writes `hasCompletedOnboarding: true` and `lastOnboardingVersion: MACRO.VERSION` to global config — the version pin means future major-version rollouts can re-trigger onboarding without also re-triggering the trust dialog.

### 2.3 Trust gate (lines 130–171)
Skipped when `process.env.CLAUBBIT` is truthy (claubbit is an internal harness environment that pre-provisions trust). Otherwise:

- **Fast path (line 135):** `checkHasTrustDialogAccepted()` consults session state + disk cascade (see §3). If `true`, the `TrustDialog` dynamic import and render are both skipped — saves ~a bundle chunk's parse time and one React commit for already-trusted repos (the common case on repeat invocations).
- **Slow path (lines 136–140):** dynamically import `./components/TrustDialog/TrustDialog.js`, render it in an `AppStateProvider`/`KeybindingSetup` wrapper via `showSetupDialog` (lines 86–92). The dialog itself either persists acceptance to disk or calls `gracefulShutdownSync(1)` on decline (see §4).
- **Post-accept (lines 142–150):** `setSessionTrustAccepted(true)` flips bootstrap state so the same-process `computeTrustDialogAccepted()` short-circuit starts returning `true` immediately. Then `resetGrowthBook(); void initializeGrowthBook();` — the **reset+reinit is the login/logout defense**: any stale GrowthBook client from a previous auth state is discarded so the fresh client picks up the new auth headers (which `sessionTrustAccepted` is itself a prerequisite for — headers are only attached post-trust).
- **Fire-and-forget prefetch (line 153):** `void getSystemContext()` warms the git/OS context cache while the user reads the next dialog.
- **MCP-json approvals (lines 156–161):** `handleMcpjsonServerApprovals(root)` — only when settings parsed cleanly (`allErrors.length === 0`); if settings have errors the user will see them in the REPL and fix them before trusting any new servers.
- **CLAUDE.md external-include approvals (lines 164–170):** `shouldShowClaudeMdExternalIncludesWarning()` → render `ClaudeMdExternalIncludesDialog` for each new `@path/outside/project` include. External includes are treated as an additional trust decision because a repo can point CLAUDE.md at arbitrary filesystem paths.

**The entire block above is inside `if (!CLAUBBIT)`, so the bootstrap ordering below runs regardless of whether the dialog was shown.**

### 2.4 Post-trust bootstrap (lines 173–190)
These run even on the fast-path (trust already accepted) and are **ordered by dependency**:

1. `void updateGithubRepoPathMapping()` — line 175. Comment explicitly calls out: "must happen AFTER trust to prevent untrusted directories from poisoning the mapping". This mapping drives teleport (directory-switch) UX; a malicious repo could otherwise register itself under a spoofed remote.
2. `if (feature('LODESTONE')) updateDeepLinkTerminalPreference()` — line 177. LODESTONE is a feature-gated deep-link terminal integration.
3. `applyConfigEnvironmentVariables()` — line 184. **Applies managed-env entries from settings** (including `env.VAR` rules from enterprise-managed or project-level settings). The comment is explicit that this is where "potentially dangerous environment variables from untrusted sources" land — this is precisely why it must follow trust.
4. `setImmediate(() => initializeTelemetryAfterTrust())` — line 190. Deferred to next tick so the OTel dynamic import resolves *after* first render, avoiding a blocking import during the pre-render microtask queue. `otelHeadersHelper` can execute shell commands, so it is trust-gated.

### 2.5 Grove policy dialog (lines 191–201)
`isQualifiedForGrove()` checks org policy. On `'escape'` the user has declined the Grove policy — we call `logEvent('tengu_grove_policy_exited')`, `gracefulShutdownSync(0)` (note: code 0, not 1 — declining policy is a normal exit, not an error), and return `false`.

### 2.6 Custom-API-key approval (lines 206–217)
```ts
if (process.env.ANTHROPIC_API_KEY && !isRunningOnHomespace()) { … }
```
`isRunningOnHomespace()` excludes a hosted environment where `ANTHROPIC_API_KEY` is deliberately preserved in env for *child* processes but ignored by Claude Code itself — short-circuits to avoid prompting on every session. For normal runs we hash-truncate the key via `normalizeApiKeyForConfig`, classify it (`'new' | 'approved' | 'rejected'` via `getCustomApiKeyStatus`), and show `ApproveApiKey` only for `'new'` — user decision is stored by short-hash so a rotated key re-prompts but a repeated key does not.

### 2.7 Bypass permissions + auto mode (lines 218–235)
- **`BypassPermissionsModeDialog`** (218–223): when `permissionMode === 'bypassPermissions'` or `--dangerously-skip-permissions` flag, and `!hasSkipDangerousModePermissionPrompt()` setting. No exit-on-decline — `done` is wired to `onAccept`, so decline is handled inside the dialog.
- **`AutoModeOptInDialog`** (224–235): gated on `feature('TRANSCRIPT_CLASSIFIER')`, `permissionMode === 'auto'`, and `!hasAutoModeOptIn()`. On decline it calls `gracefulShutdownSync(1)` via `declineExits`. Comment notes that if the server-side auto-mode gate denied access, this dialog is suppressed (showing consent for an unavailable feature is pointless) — `verifyAutoModeGateAccess` puts up an explainer instead.

### 2.8 Dev-channels dialog (lines 241–288)
Gated on `feature('KAIROS') || feature('KAIROS_CHANNELS')`. Careful cache warming: if the user passed `--channels` or `--dangerously-load-development-channels`, call `checkGate_CACHED_OR_BLOCKING('tengu_harbor')` first. Comment refs **gh#37026** — a cold disk cache defaults the gate to `false`, which would silently drop channel notifications for the whole session; blocking on the gate warms it. If channels are disabled or the user lacks OAuth, we still append dev entries to the allowlist (so `ChannelsNotice` can render the "blocked" branch with names) but skip the confirmation dialog — "accepting then seeing 'not available' in ChannelsNotice is worse than no dialog".

### 2.9 Chrome onboarding (lines 291–296)
Final dialog: `claudeInChrome && !hasCompletedClaudeInChromeOnboarding`. Marks Chrome-specific onboarding done.

Returns `onboardingShown` so the caller (`main.tsx`) can adjust first-run copy/welcome.

---

## 3. Trust persistence — `src/utils/config.ts`

The trust cascade lives in `src/utils/config.ts` and is two-tier: session memory + per-directory disk.

- **`checkHasTrustDialogAccepted()`** (lines 697–703): memoized single-directory check. Memoizes *only* the `true` result; `false` is re-checked on every call because trust can flip during a session (the dialog writes disk state that a later `computeTrustDialogAccepted` call would miss if we memoized false).
- **`computeTrustDialogAccepted()`** (lines 705–743): the actual cascade.
  - Line 709: check `getSessionTrustAccepted()` from `src/bootstrap/state.ts` (set by `setSessionTrustAccepted(true)` at `interactiveHelpers.tsx:144`).
  - Lines 718–721: check current project path (normalized).
  - Lines 726–740: walk up parent directories — if any ancestor has `hasTrustDialogAccepted: true`, the current dir is trusted. This lets `~/src/` be trusted once and propagate to all descendant repos.
- **`isPathTrusted(dir)`** (lines 752–761): for *arbitrary* directories (e.g. when the agent is asked to install/operate on a user-typed path). **Does not consult session trust or memoize** — always walks disk. Used where the caller is asking about a path that isn't `cwd`.
- **`saveCurrentProjectConfig()`** (lines 1625–1698): writes `hasTrustDialogAccepted: true` under `config.projects[absolutePath]` in the global config JSON.

**Home directory special case:** `TrustDialog.tsx` (line 175) routes home-dir acceptance to `setSessionTrustAccepted(true)` *only* — skipping `saveCurrentProjectConfig()`. This prevents the user's entire home directory from being permanently marked trusted, which would auto-trust every repo they ever clone.

`ProjectConfig.hasTrustDialogAccepted` is declared at `config.ts:111`; the default is `false` (line 138–148 `DEFAULT_PROJECT_CONFIG`).

---

## 4. TrustDialog internals — `src/components/TrustDialog/TrustDialog.tsx`

File is 290 lines; key references:

- **Line 23:** component entry `TrustDialog({ commands, onDone })`.
- **Dangerous-feature detection (lines 31–127):** the dialog inspects the current workspace for signals that *should* affect the warning copy. Each feature family it detects:
  - MCP servers from project scope (31–38)
  - hooks in settings (49–54)
  - bash in commands/permissions (58–127 — ranges for each flavor)
  - `apiKeyHelper` script (66–71)
  - AWS/GCP credential-refresh/export commands (75–90)
  - `otelHeadersHelper` (93–98)
  - `env` rules from managed env (102–107)
- **Line 134:** logs `tengu_trust_dialog_shown` with the above as fields — analytics can slice "which dangerous features appeared and trust was still accepted/declined".
- **Lines 174–177:** accept branch splits on home-vs-project as described above.
- **Line 257:** the actual prompt render — binary "Yes, I trust this folder" / "No, exit".

Decline path calls `gracefulShutdownSync(1)` (error exit — user rejected workspace).

---

## 5. Onboarding — `src/components/Onboarding.tsx`

Collects theme preference (and whatever the current onboarding UX branches into — it may render `ConsoleOAuthFlow` for logged-out users depending on flags). Terminal preference is also probed here. On done it calls back into `interactiveHelpers.tsx:completeOnboarding()` (lines 32–38) which writes `hasCompletedOnboarding: true` and `lastOnboardingVersion: MACRO.VERSION` to global config.

---

## 6. Authentication cascade — `src/utils/auth.ts`

Two parallel cascades: OAuth-token source and API-key source. They are queried independently by callers depending on the transport.

### 6.1 `getAuthTokenSource()` (auth.ts:153–206) — OAuth tokens
Priority:
1. `ANTHROPIC_AUTH_TOKEN` env var (unless managed OAuth context) — 164–166
2. `CLAUDE_CODE_OAUTH_TOKEN` env var (managed OAuth — always wins) — 168–170
3. OAuth token from file descriptor (CCR disk fallback for CI credential passing) — 173–191
4. `apiKeyHelper` from settings (unless managed OAuth) — 195–198
5. Claude.ai OAuth tokens from secure storage — 200–203
6. `none` — 205

### 6.2 `getAnthropicApiKeyWithSource()` (auth.ts:226–348) — API keys
Priority (skips OAuth entirely; used for raw-API-key clients):
1. **Bare mode** (`--bare`): only `ANTHROPIC_API_KEY` env or `apiKeyHelper` — 236–248
2. `ANTHROPIC_API_KEY` if a preferred third-party auth is active — 254–263
3. **CI mode:** FD token / env / error — 265–297
4. Approved `ANTHROPIC_API_KEY` from config (see §9) — 299–309
5. API key from file descriptor — 312–318
6. `apiKeyHelper` (with sync cache) — 321–337
7. Config or macOS keychain via `getApiKeyFromConfigOrMacOSKeychain()` — 339–342

### 6.3 `isAnthropicAuthEnabled()` (auth.ts:100–149)
Returns `false` (disabling OAuth attempts entirely) when:
- bare mode,
- any of `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` / `CLAUDE_CODE_USE_FOUNDRY` truthy — 115–118,
- user has external API key/auth token, unless in managed OAuth context — 123–146.

---

## 7. OAuth machinery — `src/services/oauth/` + `src/constants/oauth.ts`

### 7.1 Constants (`src/constants/oauth.ts`)
- Scopes (33–58): `user:inference`, `user:profile`, full set for Claude.ai subscribers (`user:profile user:inference user:sessions mcp_servers file_upload`), Console subset for API-key creation, union in `ALL_OAUTH_SCOPES`.
- Endpoints (86–93): Console auth `https://platform.claude.com/oauth/authorize`, Claude.ai auth `https://claude.com/cai/oauth/authorize`, token `https://platform.claude.com/v1/oauth/token`, roles `https://api.anthropic.com/api/oauth/claude_cli/roles`.

### 7.2 PKCE flow (`src/services/oauth/client.ts`)
- **`buildAuthUrl()` (46–105):** constructs the authorize URL with `code_challenge` / `code_challenge_method: S256` (PKCE — 72), `scope` (84), optional `orgUUID` (90–91 — enterprise SSO pre-selection), optional `login_hint` (95–96 — pre-populate email from a prior session), optional `login_method` (100–101 — `sso`|`magic_link` forced-flow).
- **`exchangeCodeForTokens()` (107–144):** authorization-code grant with PKCE verifier, POST to `TOKEN_URL`, 15 s timeout (130–133).
- **`refreshOAuthToken()` (146–274):** refresh-token grant with optional scope expansion (150–163). On success it can fetch profile (209–211) and write display name / billing type back to config (214–238).

### 7.3 Auth-code listener (`src/services/oauth/auth-code-listener.ts`)
Spawns a localhost HTTP server on an ephemeral port that receives the redirect from the browser, plucks `code` + `state`, then exchanges via `exchangeCodeForTokens()`. Standard CLI-OAuth pattern.

### 7.4 ConsoleOAuthFlow (`src/components/ConsoleOAuthFlow.tsx:54+`)
The React/Ink wrapper that drives the above. Accepts `onDone(hasProfile)`, `mode: 'login' | 'signup' | 'link'`, and `forceLoginMethod: 'claudeai' | 'console'` (enterprise override). Used both from `Onboarding` (first-run) and from `TeleportError` recovery paths.

---

## 8. Token storage — `src/utils/secureStorage/`

Platform-specific backends with a common interface:

- **macOS:** `macOsKeychainStorage.ts` — shells out to `security find-generic-password`/`add-generic-password`.
- **Fallback (Linux, non-keychain macOS):** `plainTextStorage.ts` — writes `~/.claude/.credentials.json` with mode `0o600`.

`getClaudeAIOAuthTokens()` (auth.ts:1255–1300) reads in this priority: `CLAUDE_CODE_OAUTH_TOKEN` env (inference-only, no refresh) → FD token (CCR) → secure-storage backend. `saveOAuthTokensIfNeeded()` (auth.ts:1194–1253) writes the `claudeAiOauth` blob (accessToken, refreshToken, expiresAt, scopes, subscriptionType) to the backend — skipping if not Claude.ai auth (1198–1201) or if the token is inference-only (1204–1207).

---

## 9. Custom-key approval — `src/components/ApproveApiKey.tsx`

- **Dialog flow (19–45):** accept appends truncated key to `customApiKeyResponses.approved[]`; decline appends to `customApiKeyResponses.rejected[]`.
- **Display (line 70):** `ANTHROPIC_API_KEY: sk-ant-...{truncated}` — truncation prevents the full key from being written to terminal scrollback.
- **Lookup (auth.ts:1103–1114):** `getCustomApiKeyStatus(truncated)` returns `'approved' | 'rejected' | 'new'`. `showSetupScreens` (208–217) only shows the dialog for `'new'`; `'rejected'` keys silently fall through to the next cascade step, `'approved'` keys are accepted.

---

## 10. Enterprise SSO & forced login — `src/utils/auth.ts:1923–2000`

- **`validateForceLoginOrg()` (1923–2000):**
  - Returns `{valid: true}` immediately if no `forceLoginOrgUUID` set (1936–1939).
  - Refreshes the token first so profile reads use current creds (1943).
  - Fetches `/api/oauth/profile` **uncached, server-authoritative** (1950–1968) — this is intentional: a stale local profile could pass a check that the server would reject.
  - Compares `profile.organization.uuid` to required `forceLoginOrgUUID` (1972–1974). Mismatch → env-var tokens get a copy-paste error (1977–1990), keychain tokens get a "please log in with correct org" message (1993–1999).
- **Enforcement point:** `src/cli/handlers/auth.ts:160–164` (inside `authLogin()`). Failing validation blocks login completion.

`forceLoginMethod` (`'claudeai' | 'console'`) pipes through `buildAuthUrl`'s `login_method` param so the user can't escape the mandated auth flow.

---

## 11. login / logout commands

- **`src/cli/handlers/auth.ts:authLogin()` (112–222):** reads `forceLoginMethod` + `forceLoginOrgUUID` from managed settings (130–136); fast path for `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` env (140–186) that exchanges the refresh token directly without browser flow (enterprise automation); `validateForceLoginOrg()` (160–164); marks onboarding complete (168–171).
- **`installOAuthTokens()` (50–110):** calls `performLogout()` to clear stale state (52), fetches/stores profile (55–77), `saveOAuthTokensIfNeeded()` (79), fetches user roles (91–92). For Console users it also creates an API key (95–107).
- **logout:** removes stored tokens, clears `customApiKeyResponses`, resets GrowthBook — mirror of install.

---

## 12. Token refresh & 401 handling

This is the most load-bearing runtime logic and probably deserves a dedicated task (see §14).

- **`checkAndRefreshOAuthTokenIfNeeded()` (auth.ts:1427–1562):** local-time expiry check (1453–1461); acquires a filesystem lock on `~/.claude/` to serialize refreshes across concurrent CLI processes (1484–1491); double-checks lock held before refreshing (1521–1528); calls `refreshOAuthToken()` (1531–1539); releases lock in `finally` (1559–1561).
- **`handleOAuth401Error()` (auth.ts:1360–1392):** in-flight dedup by token (same-token 401s share one refresh promise — 1363–1370); clears memoize caches (1377); re-reads tokens async; if the keychain has a *different* token, another process already refreshed and we use that (1385–1387); otherwise force-refresh bypassing local-expiry (1391).

---

## 13. Integration points with other subsystems

- **GrowthBook (services/analytics/growthbook):** auth headers only attached when `sessionTrustAccepted` is true. `resetGrowthBook()` + `initializeGrowthBook()` after trust (interactiveHelpers.tsx:149–150) is the handshake.
- **Telemetry (entrypoints/init.ts:initializeTelemetryAfterTrust):** trust-gated because `otelHeadersHelper` can shell out.
- **Managed env (utils/managedEnv.ts):** `applyConfigEnvironmentVariables()` is explicitly sequenced to trust.
- **MCP (services/mcpServerApproval):** `handleMcpjsonServerApprovals()` runs inside the trust block; MCP server specs loaded from project `.mcp.json` are otherwise one of the dangerous-feature categories detected by TrustDialog.
- **Print mode (cli/print.ts):** bypasses `showSetupScreens` entirely — `-p` is expected to be non-interactive CI with pre-provisioned auth.
- **Shutdown (utils/gracefulShutdown.ts):** invoked on multiple decline paths — `gracefulShutdownSync(0)` for Grove policy decline (logical exit), `gracefulShutdownSync(1)` for AutoMode decline and TrustDialog decline (error exit).

---

## 14. Recommendation: follow-up task flag

The OAuth refresh + 401 machinery (`auth.ts:1360–1562`) is nontrivial enough to warrant its own task:
- cross-process lock semantics on `~/.claude/`,
- in-flight dedup keyed by token value,
- the "another process already refreshed" reconciliation path,
- scope-expansion refresh,
- interaction with the `apiKeyHelper` sync cache.

Similarly, the **secure-storage backend layer** (`src/utils/secureStorage/*`) — keychain vs plaintext, the migration story when a system gains/loses keychain support — is worth a dedicated pass.

Both were in scope here for map-level coverage but neither got line-by-line treatment; flagging for a future phase-2/3 deep-dive.

---

## 15. Key data structures

- **`GlobalConfig`** (utils/config.ts): `theme`, `hasCompletedOnboarding`, `lastOnboardingVersion`, `customApiKeyResponses: { approved: string[]; rejected: string[] }`, `projects: Record<absPath, ProjectConfig>`, `oauthAccount?` (cached profile).
- **`ProjectConfig.hasTrustDialogAccepted: boolean`** — the disk half of the trust cascade (config.ts:111).
- **`claudeAiOauth`** (secure-storage blob): `{ accessToken, refreshToken, expiresAt, scopes, subscriptionType }`.
- **`setSessionTrustAccepted`** (bootstrap/state.ts): process-scope boolean, the session half.
- **`AuthTokenSource`** enum-like return of `getAuthTokenSource()`: identifies which cascade step won, used for telemetry and for deciding which refresh/401 path applies.

---

## 16. Key control-flow invariants (that must not be broken)

1. **Trust precedes env application.** `applyConfigEnvironmentVariables()` must not run before the trust dialog has resolved (or been fast-path skipped because trust is already persisted). Reordering would expose the user to `env` rules from an untrusted project.
2. **Trust precedes OAuth-dependent helpers.** `apiKeyHelper` and `otelHeadersHelper` both can execute shell commands; both are trust-gated.
3. **Trust precedes `updateGithubRepoPathMapping()`.** Explicit in the comment at line 174 — otherwise a malicious repo can poison the teleport directory map.
4. **`setSessionTrustAccepted(true)` must fire before `initializeGrowthBook()`** — the GrowthBook init reads the flag to decide whether to include auth headers.
5. **Grove decline is exit-0.** Declining policy is not an error; re-reading the dialog shouldn't appear on the next run as a crash-recovery prompt.
6. **Home directory never persists trust.** `TrustDialog.tsx:174–177` bifurcates; breaking this makes every repo the user ever clones auto-trusted.
7. **`computeTrustDialogAccepted` must re-check false.** Memoizing `false` would prevent the same-process transition from declined→accepted on a second prompt.
8. **Enterprise `forceLoginOrgUUID` uses uncached profile.** `auth.ts:1950–1968` — a cached stale profile could bypass the check.

---

*Report produced for Task #11, wave-1 phase-1. Cross-refs to Task #4 (MCP bootstrap), future task on OAuth refresh/401 flow, and Task #15 (cleanup/exit — which consumes `gracefulShutdownSync` called from decline branches here).*
