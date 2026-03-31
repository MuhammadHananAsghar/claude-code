# Remote Sessions System — Claude Code Internal Architecture

> Extracted from Claude Code npm source map leak (March 31, 2026).  
> Security research / bug bounty reference. Static analysis only — code cannot be run as-is.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture Paths (V1 vs V2)](#2-architecture-paths-v1-vs-v2)
3. [Feature Gates (GrowthBook)](#3-feature-gates-growthbook)
4. [Authentication Layers](#4-authentication-layers)
5. [OAuth Token Management](#5-oauth-token-management)
6. [Trusted Device Token (Elevated Auth)](#6-trusted-device-token-elevated-auth)
7. [Session Ingress Auth](#7-session-ingress-auth)
8. [Transport Types](#8-transport-types)
9. [WebSocketTransport](#9-websockettransport)
10. [SSETransport](#10-ssetransport)
11. [HybridTransport](#11-hybridtransport)
12. [CCRClient (Cloud Code Runtime)](#12-ccrclient-cloud-code-runtime)
13. [CONNECT-over-WebSocket Relay (Upstream Proxy)](#13-connect-over-websocket-relay-upstream-proxy)
14. [Bridge API Endpoints](#14-bridge-api-endpoints)
15. [Bridge Main Loop (V1)](#15-bridge-main-loop-v1)
16. [Environment-less Bridge Core (V2)](#16-environment-less-bridge-core-v2)
17. [Session Bootstrap (initReplBridge)](#17-session-bootstrap-initreplbridge)
18. [Session Lifecycle](#18-session-lifecycle)
19. [Worktree Provisioning](#19-worktree-provisioning)
20. [Background Remote Sessions](#20-background-remote-sessions)
21. [Permission & Policy System](#21-permission--policy-system)
22. [Message Deduplication](#22-message-deduplication)
23. [Telemetry and Activity Tracking](#23-telemetry-and-activity-tracking)
24. [Error Codes & Close Codes](#24-error-codes--close-codes)
25. [Key Constants Reference](#25-key-constants-reference)
26. [File Inventory](#26-file-inventory)
27. [Security Findings](#27-security-findings)

---

## 1. Overview

Claude Code implements a **Remote Control** (CCR — Cloud Code Runtime) system that allows the CLI running on a developer's machine or a cloud container to be driven by the claude.ai web frontend. This enables:

- **Remote-control from browser**: User opens a session in claude.ai, the CLI on their local machine or a cloud VM executes tasks
- **Background cloud sessions**: Headless CLI agents running in Anthropic-managed infrastructure
- **Mirror mode**: Local sessions mirrored to a remote environment

The system is gated behind `tengu_ccr_bridge` GrowthBook feature flag and requires a claude.ai OAuth account.

**Primary source directory:** `src/bridge/`  
**Transport layer:** `src/cli/transports/`  
**Utilities:** `src/utils/sessionIngressAuth.ts`, `src/utils/worktree.ts`  
**Upstream proxy tunnel:** `src/upstreamproxy/relay.ts`

---

## 2. Architecture Paths (V1 vs V2)

### V1 — Environment-Based (Poll/Dispatch Model)

```
claude.ai  →  Environments API  →  CLI polls /work/poll
                                    ↓
                              Spawn session process
                                    ↓
                          WebSocket/SSE transport back to session-ingress
```

- CLI registers an "environment" (`POST /v1/environments/bridge`)
- Polls for work (`GET /v1/environments/{id}/work/poll`) in a long-poll loop
- On work available: spawns a session subprocess with `WorkSecret` payload
- Session communicates back via transport (WebSocket or SSE)
- Heartbeats keep the work lease alive

**Entry point:** `src/bridge/bridgeMain.ts` (33K-line daemon)  
**Gate:** Default path when `tengu_bridge_repl_v2` is **disabled**

### V2 — Environment-less (Direct Session Path)

```
claude.ai  →  POST /v1/code/sessions  →  session created
                   ↓
           POST /v1/code/sessions/{id}/bridge  →  worker_jwt issued
                   ↓
           CCRClient (SSE read + HTTP POST write) to session-ingress
```

- No poll loop; no environment registration
- OAuth token → worker JWT exchange at session start
- Direct SSE stream + HTTP POST for bidirectional communication

**Entry point:** `src/bridge/remoteBridgeCore.ts`  
**Gate:** Enabled when `tengu_bridge_repl_v2` is **enabled**

---

## 3. Feature Gates (GrowthBook)

| Gate | File | Effect |
|------|------|--------|
| `tengu_ccr_bridge` | `bridgeEnabled.ts` | Master switch — enables/disables Remote Control entirely |
| `tengu_bridge_repl_v2` | `bridgeEnabled.ts` | Use V2 (env-less) path vs V1 (poll) |
| `tengu_bridge_repl_v2_cse_shim_enabled` | `bridgeEnabled.ts` | Retag `cse_*` IDs to `session_*` for claude.ai frontend compat |
| `tengu_sessions_elevated_auth_enforcement` | `trustedDevice.ts` | Require trusted device token for SecurityTier=ELEVATED sessions |
| `tengu_ccr_bridge_multi_session` | `bridgeMain.ts` | Allow multiple sessions per environment |
| `tengu_ccr_bundle_seed_enabled` | `remoteSession.ts` | Bundle-based deployment (no remote git required) |
| `tengu_cobalt_harbor` | `bridgeEnabled.ts` | Auto-connect to CCR on session start |
| `tengu_ccr_mirror` | `bridgeEnabled.ts` | Mirror local sessions to remote |

---

## 4. Authentication Layers

Remote sessions use a **three-layer** auth stack:

```
Layer 1: OAuth Access Token (claude.ai login)
    ↓ exchanged for
Layer 2: Session Ingress Token (JWT or session-key)
    ↓ optionally augmented by
Layer 3: Trusted Device Token (X-Trusted-Device-Token header)
```

All HTTP calls use:
```typescript
{
  Authorization: `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'anthropic-beta': 'environments-2025-11-01',
  'x-environment-runner-version': deps.runnerVersion,
  // optional:
  'X-Trusted-Device-Token': deviceToken,
}
```

---

## 5. OAuth Token Management

**File:** `src/bridge/bridgeApi.ts`

### Auto-Retry on 401

```typescript
async function withOAuthRetry<T>(
  fn: (accessToken: string) => Promise<{ status: number; data: T }>,
  context: string,
): Promise<{ status: number; data: T }> {
  const accessToken = resolveAuth()
  const response = await fn(accessToken)

  if (response.status !== 401) return response

  if (!deps.onAuth401) return response  // no refresh handler

  const refreshed = await deps.onAuth401(accessToken)
  if (refreshed) {
    const newToken = resolveAuth()
    return fn(newToken)   // single retry only
  }
  return response
}
```

- **Single retry only** — avoids refresh loops
- 401 on retry = permanent failure
- `onAuth401` callback allows injecting token refresh logic

### Proactive Refresh (initReplBridge)

```typescript
// Cross-process backoff for dead tokens
const cfg = getGlobalConfig()
if (cfg.bridgeOauthDeadExpiresAt != null &&
    (cfg.bridgeOauthDeadFailCount ?? 0) >= 3) {
  return null  // skip silently, back off
}

// Refresh if expired before attempting to connect
await checkAndRefreshOAuthTokenIfNeeded()
```

- Tracks consecutive OAuth dead states in global config
- 3 consecutive failures → 15-minute quiet period
- Proactive refresh avoids 401s at the 8-hour TTL boundary

### Token Override (Anthropic Internal)

```typescript
// src/bridge/bridgeConfig.ts
export function getBridgeAccessToken(): string | undefined {
  return getBridgeTokenOverride() ?? getClaudeAIOAuthTokens()?.accessToken
}
// CLAUDE_BRIDGE_OAUTH_TOKEN env var: hardcoded token for Anthropic dev use (ant-only)
```

---

## 6. Trusted Device Token (Elevated Auth)

**File:** `src/bridge/trustedDevice.ts`

CCR V2 sessions have `SecurityTier=ELEVATED` server-side. The server gates connection on its own flag (`sessions_elevated_auth_enforcement`). The CLI sends the token via:

```
X-Trusted-Device-Token: <token>
```

### Enrollment Flow

```typescript
// Triggered during login for ant users or when ELEVATED tier required
const response = await axios.post<{
  device_token?: string
  device_id?: string
}>(
  `${baseUrl}/api/auth/trusted_devices`,
  { display_name: `Claude Code on ${hostname()} · ${process.platform}` },
)
```

### Storage

Stored in OS-native secure storage:
- **macOS**: Keychain (`security add-generic-password`)
- **Windows**: DPAPI-encrypted file
- **Linux**: `pass` or fallback file

Reads are memoized to avoid repeated `security` subprocess spawning (~40ms per call).

### Expiry

- **Rolling 90-day expiry**
- Cleared on logout via `clearAuthRelatedCaches()`

---

## 7. Session Ingress Auth

**File:** `src/utils/sessionIngressAuth.ts`

### Token Resolution Priority

```typescript
export function getSessionIngressAuthToken(): string | null {
  // 1. Environment variable
  const envToken = process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN
  if (envToken) return envToken

  // 2. File descriptor (legacy)
  // Uses /dev/fd/{fd} on macOS, /proc/self/fd/{fd} on Linux
  // CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR

  // 3. Well-known file
  // CLAUDE_SESSION_INGRESS_TOKEN_FILE
  // Default: /home/claude/.claude/remote/.session_ingress_token
}
```

### Header Construction

```typescript
export function getSessionIngressAuthHeaders(): Record<string, string> {
  const token = getSessionIngressAuthToken()
  if (!token) return {}

  // Session keys (sk-ant-sid*) → Cookie + X-Organization-Uuid
  if (token.startsWith('sk-ant-sid')) {
    return {
      Cookie: `sessionKey=${token}`,
      'X-Organization-Uuid': process.env.CLAUDE_CODE_ORGANIZATION_UUID,
    }
  }

  // JWTs → Bearer Authorization
  return { Authorization: `Bearer ${token}` }
}
```

### Live Token Update

```typescript
// Supports in-process token refresh without restart
export function updateSessionIngressAuthToken(token: string): void {
  process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN = token
}
```

---

## 8. Transport Types

Three transports are available, selected by environment variable:

| Env Var | Transport | Read | Write |
|---------|-----------|------|-------|
| `CLAUDE_CODE_USE_CCR_V2=1` | SSETransport | SSE stream | HTTP POST |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1` | HybridTransport | WebSocket | HTTP POST |
| (default) | WebSocketTransport | WebSocket | WebSocket |

All implement a common `Transport` interface:

```typescript
interface Transport {
  connect(): Promise<void>
  write(message: StdoutMessage): Promise<void>
  close(): void
  isConnectedStatus(): boolean
  isClosedStatus(): boolean
  setOnData(callback: (data: string) => void): void
  setOnClose(callback: (closeCode?: number) => void): void
  setOnConnect?(callback: () => void): void    // WebSocket only
  setOnEvent?(callback: (event: StreamClientEvent) => void): void  // SSE only
}
```

---

## 9. WebSocketTransport

**File:** `src/cli/transports/WebSocketTransport.ts`

### Connection Constants

```typescript
const SLEEP_DETECTION_THRESHOLD_MS = 60_000  // 2× DEFAULT_MAX_RECONNECT_DELAY
const PERMANENT_CLOSE_CODES = new Set([
  1002,  // protocol error — server rejected handshake
  4001,  // session expired / not found
  4003,  // unauthorized
])
```

### Runtime Dispatch

```typescript
// Bun native WebSocket (preferred — no extra dependency)
if (typeof Bun !== 'undefined') {
  const ws = new globalThis.WebSocket(url, {
    headers,
    proxy: getWebSocketProxyUrl(url),
    tls: getWebSocketTLSOptions() || undefined,
  } as unknown as string[])
} else {
  // Node.js: ws package with proxy agent
  const { default: WS } = await import('ws')
  const ws = new WS(url, {
    headers,
    agent: getWebSocketProxyAgent(url),
    ...getWebSocketTLSOptions(),
  })
}
```

### Reconnect Logic

- **Exponential backoff** with configurable max
- **Sleep detection**: If gap between reconnects > 60s, treat as device wake-up
- **Permanent close codes**: Stop retrying immediately on 1002, 4001, 4003

---

## 10. SSETransport

**File:** `src/cli/transports/SSETransport.ts`

### Timing Constants

```typescript
const RECONNECT_BASE_DELAY_MS = 1_000
const RECONNECT_MAX_DELAY_MS = 30_000
const RECONNECT_GIVE_UP_MS = 600_000   // 10 minutes total budget
const LIVENESS_TIMEOUT_MS = 45_000     // dead if silent for 45s
const PERMANENT_HTTP_CODES = [401, 403, 404]
```

### Sequence Number Tracking

```typescript
private lastSequenceNum = 0
private seenSequenceNums = new Set<number>()

// Prune when set exceeds 1000 entries
// Keeps: nums > (lastSequenceNum - 200)
getLastSequenceNum(): number {
  return this.lastSequenceNum
}
```

Used on reconnect to resume from correct position — sends `?since={lastSequenceNum}` to the server.

### Event Format

```typescript
export type StreamClientEvent = {
  event_id: string
  sequence_num: number
  event_type: string
  source: string
  payload: Record<string, unknown>
  created_at: string
}
```

### Auth Handling

```typescript
const authHeaders = this.getAuthHeaders()
// Cookie header takes priority: drop Authorization to avoid conflict
if (authHeaders['Cookie']) {
  delete headers['Authorization']
}
```

### Write Retries

- `POST_MAX_RETRIES = 10`
- `POST_BASE_DELAY_MS = 500`, `POST_MAX_DELAY_MS = 8_000`
- **4xx (except 429)** → permanent failure, no retry
- **5xx + 429** → retry with exponential backoff

---

## 11. HybridTransport

**File:** `src/cli/transports/HybridTransport.ts`

Extends `WebSocketTransport` — reads via WebSocket, writes via HTTP POST batches.

### Batch Uploader Config

```typescript
this.uploader = new SerialBatchEventUploader<StdoutMessage>({
  maxBatchSize: 500,
  maxQueueSize: 100_000,   // memory bound
  baseDelayMs: 500,
  maxDelayMs: 8_000,
  jitterMs: 1_000,
  maxConsecutiveFailures,  // optional cap on retry storms
  onBatchDropped: (batchSize, failures) => {
    logForDiagnosticsNoPII('error', 'cli_hybrid_batch_dropped_max_failures', {
      batchSize, failures,
    })
  },
  send: batch => this.postOnce(batch),
})
```

### Stream Event Coalescing

Content deltas are buffered for up to `BATCH_FLUSH_INTERVAL_MS` before uploading. This reduces POST count for rapid streaming responses.

---

## 12. CCRClient (Cloud Code Runtime)

**File:** `src/cli/transports/ccrClient.ts`

The CCRClient wraps SSETransport and adds the full CCR V2 worker protocol:

### Class Definition

```typescript
export class CCRClient {
  private workerEpoch = 0
  private readonly heartbeatIntervalMs: number  // default: 20,000ms
  private readonly heartbeatJitterFraction: number
  private heartbeatTimer: NodeJS.Timeout | null = null
  private heartbeatInFlight = false
  private closed = false
  private consecutiveAuthFailures = 0           // max: 10 (200s window)
  private currentState: SessionState | null = null
  private readonly sessionBaseUrl: string
  private readonly sessionId: string
  private readonly http = createAxiosInstance({ keepAlive: true })
}
```

### Epoch Management

The server assigns each worker an integer `epoch`. If a newer worker connects (e.g., after CLI restart), the server bumps the epoch. The old CCRClient receives a 409 response and calls `onEpochMismatch`:

```typescript
private readonly onEpochMismatch: () => never
// Default: process.exit(1) for spawn-mode
// replBridge MUST override with graceful close — exit kills the user's REPL
```

Epoch mismatch flow:
1. Server returns HTTP 409
2. `onEpochMismatch()` called
3. V2 transport: triggers close with code 4090
4. `replBridgeTransport.ts` catches and initiates transport rebuild

### Heartbeat Protocol

```
CCRClient  →  POST /sessions/{id}/worker/heartbeat
             ←  { lease_extended: bool, state: string, ttl_seconds: 60 }
```

- **Interval:** 20 seconds (keeps alive within 60s server TTL)
- **Jitter support:** Prevents thundering herd with many workers
- **Max consecutive auth failures:** 10 (at 20s each = 200s window)
- After 10 auth failures → close with permanent error

### Worker State Machine

```typescript
type SessionState =
  | 'waiting'       // Initial state
  | 'running'       // Active task
  | 'idle'          // Task complete, awaiting next
  | 'disconnected'  // Connection lost

// Reports state via PUT /sessions/{id}/worker
```

### Stream Event Accumulator

Coalesces streaming assistant text deltas into complete snapshots. Enables mid-stream reconnects to deliver complete text, not fragments:

```typescript
private streamTextAccumulator = createStreamAccumulator()
private streamEventBuffer: SDKPartialAssistantMessage[] = []
private streamEventTimer: ReturnType<typeof setTimeout> | null = null
const STREAM_EVENT_FLUSH_INTERVAL_MS = 100
```

### Event Upload Queues

Three parallel `SerialBatchEventUploader` instances:

| Queue | maxBatchSize | maxBatchBytes | maxQueueSize |
|-------|-------------|---------------|-------------|
| Client events | 100 | 10 MB | 100,000 |
| Internal events | 100 | 10 MB | 200 |
| Delivery tracker | — | — | — |

---

## 13. CONNECT-over-WebSocket Relay (Upstream Proxy)

**File:** `src/upstreamproxy/relay.ts`

Implements a local TCP proxy that accepts HTTP CONNECT from native tools (curl, gh, kubectl) and tunnels the bytes over WebSocket to the CCR upstream proxy endpoint.

### Why WebSocket (not raw CONNECT)

> CCR ingress is GKE L7 with path-prefix routing; there's no `connect_matcher` in cdk-constructs. The session-ingress tunnel (`sessions/tunnel/v1alpha/tunnel.proto`) already uses this pattern.

### Protocol: Protobuf-over-WebSocket

Bytes are wrapped in `UpstreamProxyChunk` protobuf messages:

```protobuf
message UpstreamProxyChunk { bytes data = 1; }
```

Hand-rolled varint encoder (no proto library dependency):

```typescript
export function encodeChunk(data: Uint8Array): Uint8Array {
  const len = data.length
  const varint: number[] = []
  let n = len
  while (n > 0x7f) {
    varint.push((n & 0x7f) | 0x80)
    n >>>= 7
  }
  varint.push(n)
  const out = new Uint8Array(1 + varint.length + len)
  out[0] = 0x0a  // field 1, wire type 2 (length-delimited)
  out.set(varint, 1)
  out.set(data, 1 + varint.length)
  return out
}
```

### Authentication in CONNECT Handshake

```typescript
ws.onopen = () => {
  const head =
    `${connectLine}\r\n` +
    `Proxy-Authorization: ${authHeader}\r\n` +  // Basic auth: sessionId:token
    `\r\n`
  ws.send(encodeChunk(Buffer.from(head, 'utf8')))
  // Keepalive: ping every 30s
  st.pinger = setInterval(sendKeepalive, PING_INTERVAL_MS, ws)
}
```

Server-side: GKE L7 load balancer terminates TLS, MITMs the tunnel, injects org-configured credentials (e.g., DD-API-KEY), and forwards to real upstream.

---

## 14. Bridge API Endpoints

**File:** `src/bridge/bridgeApi.ts`

All IDs validated against `/^[a-zA-Z0-9_-]+$/` before inclusion in URLs (path traversal prevention).

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/v1/environments/bridge` | Register environment (V1) |
| GET | `/v1/environments/{id}/work/poll` | Long-poll for work (V1) |
| POST | `/v1/environments/{id}/work/{workId}/heartbeat` | Extend work lease |
| POST | `/v1/environments/{id}/work/{workId}/ack` | Acknowledge work start |
| POST | `/v1/code/sessions` | Create session (V2) |
| POST | `/v1/code/sessions/{id}/bridge` | Fetch worker JWT (V2) |
| POST | `/v1/sessions/{id}/archive` | Archive/close session |
| PATCH | `/v1/sessions/{compatId}` | Update session title |
| POST | `/v1/sessions/{id}/events` | Send permission response events |
| POST | `/v1/sessions/{id}/worker/heartbeat` | CCRClient heartbeat |
| PUT | `/v1/sessions/{id}/worker` | Report worker state |

### Work Secret Structure

The `WorkSecret` payload (base64url-encoded JSON) delivered at session dispatch:

```typescript
export type WorkSecret = {
  version: number
  session_ingress_token: string        // JWT for session operations
  api_base_url: string
  sources: Array<{
    type: string
    git_info?: { type: string; repo: string; ref?: string; token?: string }
  }>
  auth: Array<{ type: string; token: string }>  // credentials array
  claude_code_args?: Record<string, string> | null
  mcp_config?: unknown | null
  environment_variables?: Record<string, string> | null
  use_code_sessions?: boolean
}
```

### Permission Response Event

Sent when user responds to a tool permission dialog:

```typescript
export type PermissionResponseEvent = {
  type: 'control_response'
  response: {
    subtype: 'success'
    request_id: string
    response: Record<string, unknown>
  }
}
// POST /v1/sessions/{id}/events → { events: [event] }
```

---

## 15. Bridge Main Loop (V1)

**File:** `src/bridge/bridgeMain.ts`

### State Maps Per Session

```typescript
const activeSessions = new Map<string, SessionHandle>()
const sessionStartTimes = new Map<string, number>()
const sessionWorkIds = new Map<string, string>()
const sessionCompatIds = new Map<string, string>()      // cse_* → session_* shim
const sessionIngressTokens = new Map<string, string>()  // JWT per session
const sessionTimers = new Map<string, ReturnType<typeof setTimeout>>()
const completedWorkIds = new Set<string>()
const sessionWorktrees = new Map<string, {
  worktreePath: string
  worktreeBranch?: string
  gitRoot?: string
  hookBased?: boolean
}>()
const timedOutSessions = new Set<string>()
const titledSessions = new Set<string>()
```

### Token Refresh Scheduler

```typescript
const tokenRefresh = getAccessToken
  ? createTokenRefreshScheduler({
      getAccessToken,
      onRefresh: (sessionId, oauthToken) => {
        const handle = activeSessions.get(sessionId)
        if (v2Sessions.has(sessionId)) {
          // V2: trigger server-side reconnect (POST /reconnect)
          void api.reconnectSession(environmentId, sessionId)
        } else {
          // V1: push new OAuth token directly to session handle
          handle.updateAccessToken(oauthToken)
        }
      },
    })
  : undefined
```

### Session Spawn

On work available, `spawner.spawnSession(workSecret)` is called. The spawner:
1. Decodes the `WorkSecret`
2. Sets up subprocess environment (env vars, IPC)
3. Creates git worktree if required
4. Launches session process
5. Returns a `SessionHandle` with `{updateAccessToken, kill, waitForExit}`

---

## 16. Environment-less Bridge Core (V2)

**File:** `src/bridge/remoteBridgeCore.ts`

### Session Creation Flow

```typescript
// 1. Create session
const sessionId = await createCodeSession(baseUrl, accessToken, title, timeoutMs, tags)

// 2. Fetch worker credentials
const credentials = await fetchRemoteCredentials(sessionId, baseUrl, accessToken, timeoutMs)
// credentials = { worker_jwt, expires_in, api_base_url }

// 3. Register worker epoch
const epoch = await registerWorker(sessionUrl, ingressToken)

// 4. Build SSE + CCRClient transport
const transport = await createV2ReplTransport({
  sessionUrl, ingressToken, sessionId, epoch, getAuthToken,
})

// 5. Start message loop
```

### Transport Construction (V2)

```typescript
export async function createV2ReplTransport(opts): Promise<ReplBridgeTransport> {
  const sseUrl = new URL(sessionUrl)
  sseUrl.pathname = sseUrl.pathname.replace(/\/$/, '') + '/worker/events/stream'

  const sse = new SSETransport(
    sseUrl, {}, sessionId, undefined, initialSequenceNum, getAuthHeaders,
  )

  const ccr = new CCRClient(sse, new URL(sessionUrl), {
    getAuthHeaders,
    heartbeatIntervalMs: opts.heartbeatIntervalMs,
    onEpochMismatch: () => {
      ccr.close()
      sse.close()
      onCloseCb?.(4090)
      throw new Error('epoch superseded')
    },
  })

  return new ReplBridgeTransport(ccr, sessionId, opts)
}
```

### Echo Deduplication

```typescript
// Messages POSTed by CLI come back on the SSE read stream.
// BoundedUUIDSet tracks recently posted UUIDs to ignore echoes.
const recentPostedUUIDs = new BoundedUUIDSet(cfg.uuid_dedup_buffer_size)
const initialMessageUUIDs = new Set<string>()

// Separate dedup for re-delivered inbound prompts (sequence negotiation edge cases)
const recentInboundUUIDs = new BoundedUUIDSet(cfg.uuid_dedup_buffer_size)
```

---

## 17. Session Bootstrap (initReplBridge)

**File:** `src/bridge/initReplBridge.ts`

### Entitlement Check Sequence

```
1. isBridgeEnabledBlocking()       → GrowthBook gate check
2. getClaudeAIOAuthTokens()        → Must be signed in
3. waitForPolicyLimitsToLoad()     → Load org policy
4. isPolicyAllowed('allow_remote_control') → Org must allow
5. getOrganizationUUID()           → Get org context
6. checkAndRefreshOAuthTokenIfNeeded() → Proactive token refresh
7. Token expiry check              → Fail fast if expired post-refresh
```

### Session Title Derivation

```
Priority:
1. explicit initialName parameter
2. /rename command
3. Last meaningful user message (Haiku generation, async)
4. Generated slug fallback

Title regenerated at user message counts 1 and 3 (early session)
Out-of-order resolution guards prevent stale title overwrites
```

### Bootstrap Result

```typescript
type BridgeInitResult = {
  sessionId: string
  transport: ReplBridgeTransport
  archiveSession: () => Promise<void>
  updateTitle: (title: string) => Promise<void>
} | null  // null = ineligible (not an error)
```

---

## 18. Session Lifecycle

```
[initReplBridge]
      ↓
  [Entitlement checks]
      ↓
  [OAuth refresh]
      ↓
  [Session create / register environment]
      ↓
  [Worker JWT / epoch registration]
      ↓
  [Transport construction]  ←─────────────────────────────┐
      ↓                                                    │
  [Message loop running]                                   │
      ↓                            [4090 epoch mismatch]──┘
  [Task completes / session ends]  [Reconnect needed]
      ↓
  [POST /v1/sessions/{id}/archive]
      ↓
  [Transport close]
```

### Session Archive

```typescript
async function archiveBridgeSession(sessionId, opts): Promise<void> {
  // POST /v1/sessions/{id}/archive
  // 409 = already archived (idempotent)
  // Default timeout: 10s
  // gracefulShutdown allows only 1.5s
}
```

---

## 19. Worktree Provisioning

**File:** `src/utils/worktree.ts`

Each remote session can optionally run in an isolated git worktree.

### Worktree Session Type

```typescript
export type WorktreeSession = {
  originalCwd: string
  worktreePath: string
  worktreeName: string
  worktreeBranch?: string
  originalBranch?: string
  originalHeadCommit?: string
  sessionId: string
  tmuxSessionName?: string
  hookBased?: boolean
  creationDurationMs?: number
  usedSparsePaths?: boolean
}
```

### Slug Validation (Path Traversal Prevention)

```typescript
export function validateWorktreeSlug(slug: string): void {
  // MAX_WORKTREE_SLUG_LENGTH = 64
  // VALID_WORKTREE_SLUG_SEGMENT = /^[a-zA-Z0-9._-]+$/
  // Rejects "." and ".." path segments explicitly
}
```

### Git Credential Isolation

```typescript
const GIT_NO_PROMPT_ENV = {
  GIT_TERMINAL_PROMPT: '0',  // Prevent /dev/tty credential prompts
  GIT_ASKPASS: '',           // Disable askpass GUI programs
  // stdin: 'ignore' — blocks interactive prompts from hanging
}
```

### Directory Symlinking

To avoid disk bloat, `node_modules` and similar directories are symlinked rather than copied:

```typescript
async function symlinkDirectories(
  repoRootPath, worktreePath, dirsToSymlink,
): Promise<void> {
  // containsPathTraversal() check on each dir
  // Skips ENOENT (source missing) and EEXIST (already linked)
}
```

### Lifecycle Hooks

```typescript
executeWorktreeCreateHook()  // User-defined setup
executeWorktreeRemoveHook()  // User-defined teardown
hasWorktreeCreateHook()
```

---

## 20. Background Remote Sessions

**File:** `src/utils/background/remote/remoteSession.ts`

### Session Type

```typescript
export type BackgroundRemoteSession = {
  id: string
  command: string
  startTime: number
  status: 'starting' | 'running' | 'completed' | 'failed' | 'killed'
  todoList: TodoList
  title: string
  type: 'remote_session'
  log: SDKMessage[]
}
```

### Precondition Checks

Before launching a background remote session, all of these must pass:

```typescript
export type BackgroundRemoteSessionPrecondition =
  | { type: 'not_logged_in' }
  | { type: 'no_remote_environment' }
  | { type: 'not_in_git_repo' }
  | { type: 'no_git_remote' }
  | { type: 'github_app_not_installed' }
  | { type: 'policy_blocked' }
```

Eligibility check order:
1. Policy check (fail fast)
2. OAuth token (actionable error)
3. Remote environment availability
4. Git repo presence
5. Bundle seed gate logic (CCR can seed from local bundle, skips git requirement)
6. GitHub app installation check

---

## 21. Permission & Policy System

### Organization Policy Gate

```typescript
// src/bridge/initReplBridge.ts
await waitForPolicyLimitsToLoad()
if (!isPolicyAllowed('allow_remote_control')) {
  onStateChange?.('failed', "disabled by your organization's policy")
  return null
}
```

### Entitlement Requirements

| Requirement | Check |
|-------------|-------|
| Signed in to claude.ai | OAuth token present |
| `user:profile` scope | Organization UUID derivable |
| `allow_remote_control` policy | Org admin has not disabled |
| Minimum CLI version | Per-gate version requirements |
| `tengu_ccr_bridge` GrowthBook | Feature flag enabled for account |

---

## 22. Message Deduplication

**File:** `src/bridge/remoteBridgeCore.ts`

### Echo Dedup (Outbound → Inbound Loop)

When the CLI POSTs a message, the same message comes back on the SSE stream as an echo from the server. `BoundedUUIDSet` (bounded circular buffer) tracks recently posted UUIDs to ignore echoes:

```typescript
const recentPostedUUIDs = new BoundedUUIDSet(cfg.uuid_dedup_buffer_size)
// Pre-seeded with initialMessages to skip server history replay of past messages
```

### Inbound Dedup

Server may re-deliver inbound prompts during sequence number negotiation or transport swaps:

```typescript
const recentInboundUUIDs = new BoundedUUIDSet(cfg.uuid_dedup_buffer_size)
```

### SSE Sequence Number Dedup

`SSETransport` tracks `seenSequenceNums` (set, pruned at 1000 entries):
- Prune keeps entries `> (lastSequenceNum - 200)` — rolling 200-frame window
- Guards against duplicate delivery on reconnect with overlapping cursors

---

## 23. Telemetry and Activity Tracking

### Session Activity Callbacks

**File:** `src/utils/sessionActivity.ts`

```typescript
registerSessionActivityCallback(callback: () => void)
unregisterSessionActivityCallback(callback: () => void)
```

Fires on WebSocket data frame activity (excludes ping/pong). Used for idle-timeout diagnosis (e.g., Cloudflare 5-minute proxy cut-off).

### Diagnostic Tracking

**File:** `src/services/diagnosticTracking.ts`

Captures IDE diagnostics (errors/warnings) delta around code edits:

```typescript
interface Diagnostic {
  message: string
  severity: 'Error' | 'Warning' | 'Info' | 'Hint'
  range: { start: { line; character }; end: { line; character } }
  source?: string
  code?: string
}
// MAX_DIAGNOSTICS_SUMMARY_CHARS = 4000 (truncation limit)
```

Baselines captured before edits; new diagnostics diffed against baseline.

### Bridge-specific Telemetry Events

Emitted via `logForDiagnosticsNoPII()`:
- `cli_hybrid_batch_dropped_max_failures` — batch upload failed after max retries
- Transport state transitions (connecting, connected, disconnected)
- Epoch mismatch events
- Token refresh events

---

## 24. Error Codes & Close Codes

### WebSocket Close Codes

| Code | Meaning | Recovery |
|------|---------|----------|
| 1002 | Protocol error (rejected handshake) | Permanent stop |
| 4001 | Session expired / not found | Retry 3× (transient during server compaction) |
| 4003 | Unauthorized (invalid JWT) | Permanent stop |
| 4090 | Epoch mismatch (server bumped epoch) | Rebuild transport with fresh epoch |
| 4092 | SSE reconnect budget exhausted | Permanent stop |

### HTTP Error Handling

| Status | Meaning | Action |
|--------|---------|--------|
| 200/204 | Success | Continue |
| 401 | Auth failed | One retry with refreshed token, then `BridgeFatalError` |
| 403 (expired type) | Session expired | Prompt restart via `claude remote-control` |
| 403 (other) | Access denied | Org permissions error, `BridgeFatalError` |
| 404 | Not found | Remote Control not available for org |
| 409 | Already archived | Idempotent, not an error |
| 410 | Gone (expired) | Prompt restart |
| 429 | Rate limited | Throw transient error (back off) |

---

## 25. Key Constants Reference

| Constant | Value | Location | Meaning |
|----------|-------|----------|---------|
| `RECONNECT_BASE_DELAY_MS` | 1,000 ms | SSETransport | Backoff base |
| `RECONNECT_MAX_DELAY_MS` | 30,000 ms | SSETransport | Backoff cap |
| `RECONNECT_GIVE_UP_MS` | 600,000 ms (10 min) | SSETransport | Total retry budget |
| `LIVENESS_TIMEOUT_MS` | 45,000 ms | SSETransport | Dead connection threshold |
| `POST_MAX_RETRIES` | 10 | SSETransport | Write retry limit |
| `DEFAULT_HEARTBEAT_INTERVAL_MS` | 20,000 ms | CCRClient | CCR heartbeat interval |
| `MAX_CONSECUTIVE_AUTH_FAILURES` | 10 | CCRClient | Before giving up (200s window) |
| `STREAM_EVENT_FLUSH_INTERVAL_MS` | 100 ms | CCRClient | Delta coalescing window |
| `SLEEP_DETECTION_THRESHOLD_MS` | 60,000 ms | WebSocketTransport | Device wake-up detection |
| `PING_INTERVAL_MS` | 30,000 ms | relay.ts | WebSocket keepalive |
| `maxBatchSize` | 500 | HybridTransport | POST batch size |
| `maxQueueSize` | 100,000 | HybridTransport | Backpressure queue bound |
| Trusted device expiry | 90 days rolling | trustedDevice.ts | Keychain token lifetime |
| Bridge OAuth dead backoff | 15 min | initReplBridge.ts | After 3 consecutive dead tokens |

---

## 26. File Inventory

| File | Purpose |
|------|---------|
| `src/bridge/bridgeMain.ts` | V1 multi-session poll daemon |
| `src/bridge/remoteBridgeCore.ts` | V2 environment-less session core |
| `src/bridge/initReplBridge.ts` | Session bootstrap & entitlement checks |
| `src/bridge/bridgeApi.ts` | HTTP API client with OAuth retry |
| `src/bridge/bridgeConfig.ts` | Token/URL resolution |
| `src/bridge/bridgeEnabled.ts` | Feature gate checks |
| `src/bridge/createSession.ts` | Session create/archive/title |
| `src/bridge/replBridgeTransport.ts` | V1/V2 transport adapter |
| `src/bridge/replBridgeHandle.ts` | Transport lifecycle management |
| `src/bridge/trustedDevice.ts` | Elevated auth device enrollment |
| `src/bridge/workSecret.ts` | WorkSecret decode + worker registration |
| `src/cli/transports/WebSocketTransport.ts` | WebSocket transport |
| `src/cli/transports/SSETransport.ts` | SSE read + HTTP POST write |
| `src/cli/transports/HybridTransport.ts` | WebSocket read + HTTP POST write |
| `src/cli/transports/ccrClient.ts` | CCR V2 worker protocol |
| `src/upstreamproxy/relay.ts` | CONNECT-over-WebSocket tunnel |
| `src/utils/sessionIngressAuth.ts` | Token resolution (env/fd/file) |
| `src/utils/worktree.ts` | Git worktree provisioning |
| `src/utils/sessionActivity.ts` | Activity callback registry |
| `src/utils/background/remote/remoteSession.ts` | Background session eligibility |
| `src/services/diagnosticTracking.ts` | IDE diagnostic delta tracking |
| `src/remote/SessionsWebSocket.ts` | Sessions API WebSocket subscriber |

---

## 27. Security Findings

### Strengths

| # | Finding | Detail |
|---|---------|--------|
| S1 | OAuth-based auth throughout | No hardcoded credentials in transport path |
| S2 | ID allowlist validation | `validateBridgeId()` prevents path traversal in URL segments |
| S3 | Token refresh with single retry | Avoids refresh loops; permanent failure on retry 401 |
| S4 | Trusted device MFA | OS keychain storage, 90-day expiry, per-session |
| S5 | Backpressure queue bounds | `maxQueueSize: 100_000` prevents memory exhaustion |
| S6 | Echo + sequence deduplication | Guards replay attacks and duplicate delivery |
| S7 | Git credential isolation | `GIT_TERMINAL_PROMPT=0` prevents subprocess hijacking of tty |
| S8 | Worktree slug validation | Pattern + length check prevents path traversal |
| S9 | Per-session epoch tracking | Prevents split-brain between worker restarts |
| S10 | Per-session ingress tokens | Tokens not shared across sessions in multi-session mode |

### Concerns

| # | Severity | Finding | Detail |
|---|----------|---------|--------|
| C1 | HIGH | `WorkSecret` contains plaintext git credentials | `auth` array with raw tokens; `git_info.token` in base64url JSON; visible to subprocess debugger/dump |
| C2 | HIGH | `CLAUDE_CODE_SESSION_ACCESS_TOKEN` is process-wide | Single env var shared across all threads; visible to any code running in the process |
| C3 | MEDIUM | Token updated via `process.env` mutation | `updateSessionIngressAuthToken()` sets env var; no secure cleanup of old value from env |
| C4 | MEDIUM | V1 auth fallback: `CLAUDE_BRIDGE_OAUTH_TOKEN` env var | Ant-only override; if leaked to non-ant users through code path confusion, grants full bridge access |
| C5 | MEDIUM | CCR upstream proxy MITM by design | The relay.ts proxy is explicitly designed for server-side TLS interception and credential injection; org-level control but architectural risk |
| C6 | MEDIUM | File descriptor token source (`/dev/fd/{fd}`) | Readable by process owner until GC; FD number visible in `/proc/{pid}/fd/` on Linux |
| C7 | LOW | Sequence number set grows to 1000 before pruning | Large burst could temporarily inflate memory; not bounded per-connection |
| C8 | LOW | Session title derived from Haiku model call | Haiku-generated titles sent to external API before user-visible output; minor privacy leak of early context |
| C9 | LOW | 10 consecutive CCR auth failures before disconnect | 200-second window during which auth errors are silently accumulated |
| C10 | INFO | Epoch supersession exits process in spawn-mode | Default `onEpochMismatch = process.exit(1)`; acceptable in headless spawn but must be overridden in REPL |
