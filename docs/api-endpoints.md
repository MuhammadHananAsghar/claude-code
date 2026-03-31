# API Endpoints — Claude Code

> Extracted from Claude Code npm source map leak (March 31, 2026).  
> Security research / bug bounty reference. Static analysis only.

---

## Base URLs

| Identifier | URL | Used For |
|-----------|-----|---------|
| `BASE_API_URL` | `https://api.anthropic.com` | Default — all production endpoints |
| `ANTHROPIC_BASE_URL` | env var override | Redirect all API calls (e.g., staging, proxies) |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | env var override | OAuth only — restricted to FedStart deployments |
| `TEAM_MEMORY_SYNC_URL` | env var override | Team memory sync endpoint |

---

## Authentication Methods

| Method | Header / Field | Used With |
|--------|---------------|----------|
| OAuth 2.0 Bearer | `Authorization: Bearer {accessToken}` | All authenticated endpoints |
| API Key | `x-api-key: sk-ant-...` | Bootstrap, inference (console users) |
| Worker JWT | `Authorization: Bearer {worker_jwt}` | Session ingress, CCR worker endpoints |
| Session Key | `Cookie: sessionKey={sk-ant-sid...}` | WebSocket connections |
| Trusted Device | `X-Trusted-Device-Token: {token}` | CCR elevated-security sessions |

OAuth scopes requested during login:
- `user:inference` — send messages to the API
- `user:profile` — read org/account info
- `user:sessions:claude_code` — create/manage remote sessions
- `user:mcp_servers` — access MCP server registry
- `user:file_upload` — upload files via OAuth path

---

## Common Headers

| Header | Example Value | Purpose |
|--------|--------------|---------|
| `anthropic-version` | `2023-06-01` | API version pin |
| `anthropic-beta` | see table below | Feature flag / beta gating |
| `x-organization-uuid` | `{uuid}` | Org scoping (required for most auth'd calls) |
| `Last-Uuid` | `{uuid}` | Optimistic concurrency for session ingress |
| `Last-Event-ID` | `{sequenceNum}` | SSE stream resumption cursor |
| `User-Agent` | `Claude Code/X.Y.Z (platform)` | Standard user agent |
| `x-environment-runner-version` | `{version}` | Bridge runner version tag |

### `anthropic-beta` Values

| Value | Unlocks |
|-------|---------|
| `oauth-2025-04-20` | OAuth profile, bootstrap, team memory, policy endpoints |
| `files-2025-04-14` | Files API (upload/download/list) |
| `ccr-byoc-2025-07-29` | Cloud Code Runner session management |
| `mcp-2025-04-20` | MCP server registry |
| `environments-2025-11-01` | Bridge environments API (V1 remote control) |

---

## Endpoints

### OAuth & Authentication

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/v1/oauth/token` | `src/auth/oauth.ts` | Exchange auth code or refresh token. Body: `{ grant_type, code, redirect_uri, client_id, code_verifier, state, expires_in? }`. Response: `{ access_token, refresh_token, expires_in, scope }` |
| GET | `/api/oauth/profile` | `src/auth/oauth.ts` | Fetch authenticated user profile and org info |
| POST | `/api/oauth/claude_cli/create_api_key` | `src/auth/oauth.ts` | Create a CLI API key tied to the OAuth session |
| GET | `/api/oauth/claude_cli/roles` | `src/auth/oauth.ts` | Fetch user roles and entitlements |
| POST | `/api/auth/trusted_devices` | `src/bridge/trustedDevice.ts` | Enroll trusted device for elevated-auth CCR sessions. Body: `{ display_name }`. Response: `{ device_token, device_id }` |

---

### Inference

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/v1/messages` | `src/utils/api.ts` | Send a message to Claude (streaming or non-streaming). Standard Anthropic Messages API |
| POST | `/v1/messages/count_tokens` | `src/utils/api.ts` | Count tokens for a given messages payload without generating a response |

---

### Bootstrap & Configuration

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/claude_cli/bootstrap` | `src/utils/bootstrap.ts` | Fetch CLI boot config: additional model options, client data, policy overrides. Auth: Bearer or `x-api-key` |
| GET | `/api/claude_code/organizations/{orgUUID}/policy_limits` | `src/utils/policyLimits.ts` | Fetch org-level policy limits (allowed tools, rate limits, feature toggles) |
| GET | `/api/claude_code/organizations/{orgUUID}/remote_managed_settings` | `src/utils/remoteSettings.ts` | Fetch settings pushed from org admin |

---

### Files API

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/v1/files` | `src/services/files.ts` | Upload a file. Form-data body. Response: `{ id, filename, ... }`. Beta: `files-2025-04-14` |
| GET | `/v1/files/{fileId}/content` | `src/services/files.ts` | Download file content by ID. Beta: `files-2025-04-14` |
| GET | `/v1/files` | `src/services/files.ts` | List files. Query: `?after_created_at={timestamp}`. Response: `{ data: File[] }`. Beta: `files-2025-04-14` |
| POST | `/api/oauth/file_upload` | `src/tools/BriefTool/` | Upload file via OAuth path (Brief tool). Form-data. Response: ChatMessage file schema |
| GET | `/api/oauth/files/{uuid}/content` | `src/bridge/bridgeApi.ts` | Fetch attached file from bridge session by UUID |

---

### Remote Sessions (CCR V1 — Environment-based)

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/v1/environments/bridge` | `src/bridge/bridgeApi.ts` | Register a bridge environment. Body: `{ machine_name, directory, branch, git_repo_url, max_sessions, metadata, environment_id? }`. Response: `{ environment_id, environment_secret }` |
| GET | `/v1/environments/{id}/work/poll` | `src/bridge/bridgeApi.ts` | Long-poll for pending work. Query: `?reclaim_older_than_ms`. Response: `WorkResponse \| null` |
| POST | `/v1/environments/{id}/work/{workId}/heartbeat` | `src/bridge/bridgeApi.ts` | Extend work lease. Response: `{ lease_extended, state, ttl_seconds }` |
| POST | `/v1/environments/{id}/work/{workId}/ack` | `src/bridge/bridgeApi.ts` | Acknowledge work start |
| DELETE | `/v1/environments/{id}` | `src/bridge/bridgeApi.ts` | Deregister environment |

---

### Remote Sessions (CCR V2 — Environment-less)

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/v1/code/sessions` | `src/bridge/createSession.ts` | Create a code session (no environment_id). Body: `{ title, events, session_context: { sources, outcomes, model }, source, permission_mode? }`. Beta: `ccr-byoc-2025-07-29` |
| POST | `/v1/code/sessions/{id}/bridge` | `src/bridge/remoteBridgeCore.ts` | Exchange OAuth token for worker JWT. Response: `{ worker_jwt, expires_in, api_base_url }` |
| PUT | `/v1/code/sessions/{id}/teleport-events` | `src/remote/teleport.ts` | Append transcript message. Header: `Last-Uuid: {uuid}` for idempotency |
| GET | `/v1/code/sessions/{id}/teleport-events` | `src/remote/teleport.ts` | Fetch transcript events. Response: `{ data: TranscriptMessage[] }` |

---

### Session Management (Shared)

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/v1/sessions/{id}` | `src/bridge/bridgeApi.ts` | Fetch session details. Response: `{ environment_id, title, ... }` |
| PATCH | `/v1/sessions/{id}` | `src/bridge/createSession.ts` | Update session title. Body: `{ title }` |
| POST | `/v1/sessions/{id}/archive` | `src/bridge/createSession.ts` | Archive (close) session. 409 = already archived (idempotent) |
| POST | `/v1/sessions/{id}/events` | `src/bridge/bridgeApi.ts` | Send permission response event. Body: `{ events: [{ type: 'control_response', response: { subtype, request_id, response } }] }` |
| POST | `/v1/sessions/{id}/reconnect` | `src/bridge/bridgeApi.ts` | Trigger server-side token re-dispatch for V2 sessions after OAuth refresh |

---

### Session Ingress (Worker-side Transcript Logging)

| Method | Path | File | Description |
|--------|------|------|-------------|
| PUT | `/v1/session_ingress/session/{sessionId}` | `src/remote/sessionIngress.ts` | Append message to session transcript. Auth: worker JWT. Header: `Last-Uuid: {uuid}` |

---

### CCR Worker Protocol

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/worker/register` (relative to sessionUrl) | `src/bridge/workSecret.ts` | Register as a worker, obtain epoch. Response: `{ worker_epoch }` |
| POST | `{sessionUrl}/worker/heartbeat` | `src/cli/transports/ccrClient.ts` | CCR heartbeat every 20s. Response: `{ lease_extended, state, ttl_seconds: 60 }` |
| PUT | `{sessionUrl}/worker` | `src/cli/transports/ccrClient.ts` | Report worker state (`waiting` \| `running` \| `idle` \| `disconnected`) |

---

### Upstream Proxy (CCR Tunnel)

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/v1/code/upstreamproxy/ca-cert` | `src/upstreamproxy/` | Fetch PEM CA certificate for mTLS tunnel |
| WebSocket | `wss://…/v1/code/upstreamproxy/ws` | `src/upstreamproxy/relay.ts` | CONNECT-over-WebSocket tunnel. First frame carries `CONNECT` line + `Proxy-Authorization: Basic {sessionId:token}` |

---

### WebSocket Connections

| Method | URL | File | Description |
|--------|-----|------|-------------|
| WebSocket | `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe?organization_uuid={orgUuid}` | `src/remote/SessionsWebSocket.ts` | Subscribe to session event stream. First message: `{ type: 'auth', credential: { type: 'oauth', token } }` |
| WebSocket | `{sessionUrl}/worker/events/stream` (derived from sessionUrl) | `src/cli/transports/SSETransport.ts` | CCR V2 SSE event stream for worker. Auth: Bearer JWT |

---

### Account & Settings

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/oauth/account/settings` | `src/utils/grove.ts` | Fetch account settings. Response: `{ grove_enabled, grove_notice_viewed_at }` |
| POST | `/api/oauth/account/grove_notice_viewed` | `src/utils/grove.ts` | Mark Grove (web search privacy) notice as viewed |
| PATCH | `/api/oauth/account/settings` | `src/utils/grove.ts` | Update account settings. Body: `{ grove_enabled: boolean }` |
| POST | `/api/claude_code_grove` | `src/utils/grove.ts` | Send Grove configuration telemetry |

---

### Settings Sync

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/claude_code/settings?repo={owner/repo}` | `src/utils/settingsSync.ts` | Fetch synced settings for a repo |
| PUT | `/api/claude_code/settings?repo={owner/repo}` | `src/utils/settingsSync.ts` | Upload settings for a repo |

---

### Team Memory

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/claude_code/team_memory?repo={owner/repo}` | `src/utils/teamMemory.ts` | Fetch team memory entries. Query: `view=hashes` for checksum-only. Response: `{ entryChecksums, entries }`. Beta: `oauth-2025-04-20` |
| PUT | `/api/claude_code/team_memory?repo={owner/repo}` | `src/utils/teamMemory.ts` | Upload / upsert team memory entries. Beta: `oauth-2025-04-20` |

---

### MCP (Model Context Protocol)

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/v1/mcp_servers?limit=1000` | `src/services/mcp/officialRegistry.ts` | List official MCP servers. Response: `{ data: MCPServer[] }`. Beta: `mcp-2025-04-20` |
| ANY | `https://mcp-proxy.anthropic.com/v1/mcp/{server_id}` | `src/services/mcp/client.ts` | Proxy requests to MCP servers via Anthropic relay. Auth: OAuth Bearer |

---

### GitHub Integration

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/oauth/organizations/{orgUUID}/code/repos/{owner}/{repo}` | `src/utils/github.ts` | Check if org has access to a GitHub repo |
| POST | `/api/oauth/organizations/{orgUUID}/sync/github/auth` | `src/utils/github.ts` | Initiate GitHub OAuth sync. Body: `{ action: 'authorize' \| 'revoke' }` |

---

### Admin Requests

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/oauth/organizations/{orgUUID}/admin_requests` | `src/utils/adminRequests.ts` | List pending admin requests |
| GET | `/api/oauth/organizations/{orgUUID}/admin_requests/me?request_type={type}` | `src/utils/adminRequests.ts` | Get current user's admin request status |
| GET | `/api/oauth/organizations/{orgUUID}/admin_requests/eligibility?request_type={type}` | `src/utils/adminRequests.ts` | Check if user is eligible to submit an admin request |

---

### Referrals & Guest Passes

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/oauth/organizations/{orgUUID}/referral/eligibility?campaign={campaign}` | `src/utils/referral.ts` | Check referral eligibility. Default campaign: `claude_code_guest_pass`. Response: `{ eligible, remaining_passes, referrer_reward }` |
| GET | `/api/oauth/organizations/{orgUUID}/referral/redemptions?campaign={campaign}` | `src/utils/referral.ts` | Fetch redemption history |

---

### Usage & Quota

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/api/oauth/usage` | `src/utils/usage.ts` | Fetch usage statistics for current user |
| GET | `/api/oauth/organizations/{orgUUID}/overage_credit_grant` | `src/utils/billing.ts` | Fetch overage credit grant info |
| GET | `/v1/ultrareview/quota` | `src/utils/ultrareview.ts` | Claude Code on Web review quota. Response: `{ reviews_used, reviews_limit, reviews_remaining, is_overage }` |

---

### Remote Triggers (Scheduled Tasks)

| Method | Path | File | Description |
|--------|------|------|-------------|
| GET | `/v1/code/triggers` | `src/utils/triggers.ts` | List configured remote agent triggers |
| POST | `/v1/code/triggers` | `src/utils/triggers.ts` | Create a trigger |
| POST | `/v1/code/triggers/{trigger_id}/run` | `src/utils/triggers.ts` | Execute a trigger manually |

---

### Telemetry & Analytics

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/api/event_logging/batch` | `src/utils/telemetry.ts` | Batch first-party analytics events. Body: `{ events: [{ event_type, event_data }] }`. Path configurable via `tengu_1p_event_batch_config.path` |

---

### Health Checks

| Method | Path | File | Description |
|--------|------|------|-------------|
| POST | `/api/hello` | `src/utils/health.ts` | API health check. Response: `{ status: 'ok' }` |
| GET | `/api/oauth/hello` | `src/auth/oauth.ts` | OAuth endpoint health check |

---

## Third-Party Endpoints

| Service | Method | URL | File | Description |
|---------|--------|-----|------|-------------|
| **Datadog** | POST | `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` | `src/bridge/` | Chrome bridge event logging. Header: `DD-API-KEY: pubbbf48e6d78dae54bceaa4acf463299bf` (public ingest key) |
| **GitHub Raw** | GET | `https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md` | `src/utils/changelog.ts` | Fetch release notes / changelog |
| **Google Cloud Storage** | GET | `https://storage.googleapis.com/claude-code-dist-86c565f3-f756-42ad-8dfa-d59b1c096819/claude-code-releases/{channel}` | `src/utils/updater.ts` | Binary version metadata for auto-update |
| **Artifactory (ant-internal)** | GET | `https://artifactory.infra.ant.dev/artifactory/api/npm/npm-all/` | `src/utils/updater.ts` | Internal npm registry for Anthropic employees |
| **GrowthBook** | SDK | (client-side SDK init) | `src/utils/featureFlags.ts` | Runtime feature flags and A/B experiments |

---

## Browser / Claude.ai Endpoints

Used by the Chrome extension integration:

| URL | Purpose |
|-----|---------|
| `https://claude.ai/chrome` | Chrome extension download / setup page |
| `https://clau.de/chrome/tab/` | Tab communication bridge |
| `https://clau.de/chrome/reconnect` | Reconnection handler for Chrome bridge |

---

## Endpoint Count Summary

| Category | Count |
|----------|-------|
| OAuth & Auth | 5 |
| Inference (Messages) | 2 |
| Bootstrap & Config | 3 |
| Files API | 5 |
| Remote Sessions V1 (Environments) | 5 |
| Remote Sessions V2 (Code Sessions) | 4 |
| Session Management (Shared) | 5 |
| Session Ingress | 1 |
| CCR Worker Protocol | 3 |
| Upstream Proxy / Tunnel | 2 |
| WebSocket Connections | 2 |
| Account & Settings | 4 |
| Settings Sync | 2 |
| Team Memory | 2 |
| MCP | 2 |
| GitHub Integration | 2 |
| Admin Requests | 3 |
| Referrals & Guest Passes | 2 |
| Usage & Quota | 3 |
| Remote Triggers | 3 |
| Telemetry | 1 |
| Health Checks | 2 |
| **Third-party** | **5** |
| **Total** | **68** |
