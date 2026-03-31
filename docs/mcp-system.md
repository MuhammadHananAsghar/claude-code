# Claude Code MCP System — Deep Dive

> Based on source analysis of the leaked npm source map extraction (March 31, 2026).
> All types, constants, and logic come directly from `src/services/mcp/`, `src/tools/MCPTool/`, `src/tools/McpAuthTool/`, `src/plugins/`, `src/entrypoints/mcp.ts`, and related files.

---

## Table of Contents

1. [Overview](#overview)
2. [Transport Types](#transport-types)
3. [MCP Server Config Schemas](#mcp-server-config-schemas)
4. [Server Connection States](#server-connection-states)
5. [Connection Lifecycle](#connection-lifecycle)
6. [Transport Layer Details](#transport-layer-details)
7. [Tool Registration & Naming](#tool-registration--naming)
8. [Tool Execution Flow](#tool-execution-flow)
9. [Resource System](#resource-system)
10. [Prompt/Command System](#promptcommand-system)
11. [MCP Skills (Feature-Gated)](#mcp-skills-feature-gated)
12. [Batch Connection Orchestration](#batch-connection-orchestration)
13. [Caching Architecture](#caching-architecture)
14. [Reconnection & Error Recovery](#reconnection--error-recovery)
15. [OAuth & Authentication](#oauth--authentication)
16. [McpAuthTool — OAuth in Practice](#mcpauthto--oauth-in-practice)
17. [Elicitation System](#elicitation-system)
18. [Plugin System & MCP Integration](#plugin-system--mcp-integration)
19. [MCPB / DXT Bundle Format](#mcpb--dxt-bundle-format)
20. [Permission & Enterprise Controls](#permission--enterprise-controls)
21. [Configuration Files & Scopes](#configuration-files--scopes)
22. [Official MCP Registry](#official-mcp-registry)
23. [Claude Code as MCP Server](#claude-code-as-mcp-server)
24. [MCPTool — The Template Tool](#mcptool--the-template-tool)
25. [Timeouts Reference](#timeouts-reference)
26. [Full Connection Flow Diagram](#full-connection-flow-diagram)
27. [Security Research Notes](#security-research-notes)

---

## Overview

MCP (Model Context Protocol) is the protocol that allows Claude Code to connect to external servers that provide additional **tools**, **resources**, and **prompts**. It is based on JSON-RPC 2.0 over multiple transports.

Claude Code is simultaneously:
- An **MCP client** — connects to external MCP servers (GitHub, Playwright, databases, etc.)
- An **MCP server** — exposes itself via a direct-connect HTTP/WebSocket interface for SDK integration

Key facts:
- **6 transport types**: stdio, SSE, HTTP, WebSocket, SDK in-process, Claude.ai proxy
- **5 server connection states**: connected, failed, needs-auth, pending, disabled
- **3 levels of caching**: connection cache, tools/resources/prompts cache (LRU size 20), auth cache (15-min TTL)
- **3 timeout tiers**: connection (30s), request (60s), tool call (~27.8 hours)
- **Adaptive concurrency**: 3 simultaneous local connections, 20 remote

---

## Transport Types

**Source**: `src/services/mcp/types.ts`

```typescript
export type Transport =
  | 'stdio'         // Subprocess with stdin/stdout pipes
  | 'sse'           // Server-Sent Events (HTTP long-poll)
  | 'sse-ide'       // SSE from IDE extension (no auth required)
  | 'http'          // Streamable HTTP (POST + SSE fallback)
  | 'ws'            // WebSocket
  | 'ws-ide'        // WebSocket from IDE extension
  | 'sdk'           // In-process via SDK control channel
  | 'claudeai-proxy'// Routed through claude.ai proxy
```

### Config Scope

```typescript
export type ConfigScope =
  | 'local'       // ~/.mcp.json or .mcp.json (private)
  | 'user'        // ~/.claude/settings.json mcpServers
  | 'project'     // .mcp.json at project root
  | 'dynamic'     // Plugin-provided at runtime
  | 'enterprise'  // Admin-managed
  | 'claudeai'    // From claude.ai connector catalog
  | 'managed'     // Managed path (enterprise)
```

---

## MCP Server Config Schemas

**Source**: `src/services/mcp/types.ts`

All configs are Zod-validated:

```typescript
// Subprocess — most common for local servers
McpStdioServerConfig = {
  type: 'stdio'
  command: string          // Executable path or command
  args?: string[]          // CLI arguments
  env?: Record<string, string>  // Environment variables (${VAR} expansion supported)
}

// Server-Sent Events
McpSSEServerConfig = {
  type: 'sse'
  url: string              // SSE endpoint URL
  headers?: Record<string, string>
  headersHelper?: string   // Shell command that outputs JSON headers
  oauth?: McpOAuthConfig
}

// Streamable HTTP (newer MCP spec)
McpHTTPServerConfig = {
  type: 'http'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: McpOAuthConfig
}

// WebSocket
McpWebSocketServerConfig = {
  type: 'ws'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
}

// In-process via SDK control bridge
McpSdkServerConfig = {
  type: 'sdk'
  name: string             // SDK server identifier
}

// Claude.ai connector proxy
McpClaudeAIProxyServerConfig = {
  type: 'claudeai-proxy'
  url: string              // https://mcp-proxy.anthropic.com/v1/mcp/{server_id}
  id: string               // Connector ID from claude.ai
}

// IDE extension servers (no auth)
McpSSEIDEServerConfig = {
  type: 'sse-ide'
  url: string
  ideName: string
  ideRunningInWindows?: boolean
}

McpWebSocketIDEServerConfig = {
  type: 'ws-ide'
  url: string
  ideName: string
  authToken?: string
  ideRunningInWindows?: boolean
}
```

### OAuth Config

```typescript
export type McpOAuthConfig = {
  clientId?: string
  callbackPort?: number
  authServerMetadataUrl?: string   // Must be https://
  xaa?: boolean                    // Cross-App Access (SEP-990)
}
```

---

## Server Connection States

**Source**: `src/services/mcp/types.ts`

Each MCP server is in exactly one state:

```typescript
type MCPServerConnection =
  | ConnectedMCPServer    // Active, usable
  | FailedMCPServer       // Connection error
  | NeedsAuthMCPServer    // Got 401/403
  | PendingMCPServer      // Connecting / will retry
  | DisabledMCPServer     // Explicitly disabled in settings

// Active connection
type ConnectedMCPServer = {
  type: 'connected'
  name: string
  client: Client                   // @modelcontextprotocol/sdk Client instance
  capabilities: ServerCapabilities // tools?, resources?, prompts?, roots?, elicitation?
  serverInfo?: { name: string; version: string }
  instructions?: string            // Server-provided system prompt addition
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}

// Failed connection
type FailedMCPServer = {
  type: 'failed'
  name: string
  config: ScopedMcpServerConfig
  error?: string
}

// Needs OAuth
type NeedsAuthMCPServer = {
  type: 'needs-auth'
  name: string
  config: ScopedMcpServerConfig
}

// In-progress / retry scheduled
type PendingMCPServer = {
  type: 'pending'
  name: string
  config: ScopedMcpServerConfig
  reconnectAttempt?: number
  maxReconnectAttempts?: number
}

// Turned off
type DisabledMCPServer = {
  type: 'disabled'
  name: string
  config: ScopedMcpServerConfig
}
```

---

## Connection Lifecycle

**Source**: `src/services/mcp/client.ts` (~2700 lines)

### Step 1 — Transport Selection & Creation

```
config.type === 'sse'
  → SSEClientTransport(url, { fetch: withAuth, timeout })

config.type === 'http'
  → StreamableHTTPClientTransport(url, { fetch: withTimeout })
  → Sends Accept: 'application/json, text/event-stream'

config.type === 'ws'
  → WebSocketTransport(url, { headers })

config.type === 'stdio'
  → StdioClientTransport({ command, args, env: { ...process.env, ...config.env } })
  → Spawns subprocess

config.type === 'sdk'
  → SdkControlClientTransport  (CLI↔SDK bridge via stdout/control channel)

config.type === 'sse-ide' / 'ws-ide'
  → SSEClientTransport / WebSocketTransport (no auth required)

In-process (Chrome, Computer Use)
  → InProcessTransport linked pair (client side + server side, same process)
  → Saves ~325 MB vs subprocess spawn
```

### Step 2 — Client Initialization

```typescript
const client = new Client(
  { name: 'claude-code', version: MACRO.VERSION },
  {
    capabilities: {
      roots: {},          // Can provide workspace roots
      elicitation: {},    // Can handle elicitation requests from servers
    },
  },
)

// Register roots handler — tells server where the workspace is
client.setRequestHandler(ListRootsRequestSchema, async () => ({
  roots: [{ uri: `file://${getOriginalCwd()}` }],
}))

// Default elicitation handler (returns 'cancel' unless overridden)
client.setRequestHandler(ElicitRequestSchema, async () => ({ action: 'cancel' }))
```

### Step 3 — Connect with Timeout

```typescript
// Race: connect vs 30-second timeout
const connectionTimeout = getConnectionTimeoutMs()  // MCP_TIMEOUT env or 30000ms

await Promise.race([
  client.connect(transport),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Connection timeout')), connectionTimeout)
  ),
])
```

### Step 4 — Capability Extraction

After connection:
- Extract `capabilities` (tools, resources, prompts, roots, elicitation)
- Extract `serverInfo` (name, version)
- Extract `instructions` (optional server-provided system prompt addon)
- Extract `protocolVersion`

### Step 5 — Error Handlers

```typescript
client.onerror = (error) => {
  // Track connection issues
  // Detect session expiry: HTTP 404 + JSON-RPC -32001 → McpSessionExpiredError
  // Terminal errors trigger reconnect:
  //   ECONNRESET, ETIMEDOUT, EPIPE, EHOSTUNREACH, ECONNREFUSED
  //   ENOTFOUND, ENETUNREACH, ECONNABORTED, EADDRNOTAVAIL
}

client.onclose = () => {
  // Clear memoization caches → forces fresh connection on next tool call
  clearServerCache(name, config)
}
```

### Step 6 — Cleanup Registration

```typescript
// For stdio servers: graceful shutdown sequence
cleanup = async () => {
  process.kill(childProcess.pid, 'SIGINT')
  await sleep(100)
  process.kill(childProcess.pid, 'SIGTERM')
  await sleep(400)
  process.kill(childProcess.pid, 'SIGKILL')
  // Remove stderr listeners (prevent memory leaks)
  // Close client and transport
}
// Registered with session cleanup registry
```

---

## Transport Layer Details

### Stdio — Subprocess Lifecycle

- Spawns child process with merged env: `{ ...process.env, ...config.env }`
- `${VAR}` substitution applied to both `command` and `env` values
- Communication via stdin/stdout pipes (MCP JSON-RPC messages)
- Stderr forwarded to Claude Code logs
- Shutdown: SIGINT → 100ms → SIGTERM → 400ms → SIGKILL

### SSE / HTTP — Timeout Wrapper

```typescript
function wrapFetchWithTimeout(innerFetch, timeoutMs) {
  return async (url, init) => {
    if (init?.method === 'GET') return innerFetch(url, init)  // Skip for SSE streams
    const abortController = new AbortController()  // Fresh per-request (avoids stale signal bug)
    const timer = setTimeout(() => abortController.abort(), timeoutMs)
    try {
      return await innerFetch(url, { ...init, signal: abortController.signal })
    } finally {
      clearTimeout(timer)
    }
  }
}
```

### SDK Control Transport

```
CLI process                           SDK consumer
────────────                          ────────────
SdkControlClientTransport             SdkControlServerTransport
  ↓ serialize request                   ↑ deserialize
  ↓ write to stdout control channel     ↑ read from callback
  ↑ read response from callback         ↓ write response
```

No socket — uses the existing CLI/SDK structured IO control channel.

### In-Process Transport

```typescript
// InProcessTransport: linked pair
const [clientTransport, serverTransport] = InProcessTransport.createLinkedPair()
// client uses clientTransport, server uses serverTransport
// Both run in same V8 isolate — zero IPC overhead
```

---

## Tool Registration & Naming

**Source**: `src/services/mcp/client.ts` — `fetchToolsForClient()`

### Naming Convention

```
mcp__{normalized_server_name}__{tool_name}

Examples:
  Server "github" + tool "create_pull_request"
  → mcp__github__create_pull_request

  Server "my-server" + tool "do-thing"
  → mcp__my_server__do_thing   (hyphens → underscores)
```

### Normalization

```typescript
function normalizeNameForMCP(serverName: string): string {
  return serverName.replace(/[\s-]/g, '_')  // spaces and hyphens → underscores
}

function mcpInfoFromString(toolName: string): { serverName: string; toolName?: string } | null {
  // Parses "mcp__server__tool" → { serverName: 'server', toolName: 'tool' }
  // Parses "mcp__server" → { serverName: 'server', toolName: undefined }
}
```

### Tool Object Construction

Each MCP tool is wrapped into a Claude Code `Tool` object:

```typescript
{
  name: `mcp__${normalizedServer}__${tool.name}`,
  isMcp: true,
  inputJSONSchema: tool.inputSchema,  // Passthrough from MCP server (JSON Schema, not Zod)
  description: () => tool.description || '',
  prompt: () => PROMPT,
  isReadOnly: () => tool.annotations?.readOnlyHint ?? false,
  isDestructive: () => tool.annotations?.destructiveHint ?? false,
  isOpenWorld: () => tool.annotations?.openWorldHint ?? true,

  async call(input, context, canUseTool, parentMessage, onProgress) {
    const freshClient = await ensureConnectedClient(name, config)
    return callMCPToolWithUrlElicitationRetry(freshClient, tool.name, input, ...)
  },

  checkPermissions: () => ({ behavior: 'passthrough' }),  // Deferred to security classifier
}
```

### Tool Annotations (from MCP spec)

MCP servers can annotate tools with behavioral hints:

| Annotation | Claude Code field | Default |
|-----------|------------------|---------|
| `readOnlyHint` | `isReadOnly()` | `false` |
| `destructiveHint` | `isDestructive()` | `false` |
| `openWorldHint` | `isOpenWorld()` | `true` |

---

## Tool Execution Flow

**Source**: `src/services/mcp/client.ts` — `callMCPToolWithUrlElicitationRetry()`

```
SkillTool/model invokes mcp__server__tool
          │
          ▼
ensureConnectedClient(name, config)
→ fetch fresh connection (or use cached)
          │
          ▼
client.request({
  method: 'tools/call',
  params: { name: originalToolName, arguments: input }
}, CallToolResultSchema, { timeout: MCP_TOOL_TIMEOUT_MS })
          │
          ├─ Success → transform result content
          │   ├─ Text blocks → string
          │   ├─ Image blocks → persist to disk, return path
          │   └─ Audio blocks → persist to disk, return path
          │
          ├─ McpSessionExpiredError (404 + -32001)
          │   └─ Retry ONCE with fresh connection
          │
          ├─ MCP error -32042 (URL elicitation needed)
          │   └─ callMCPToolWithUrlElicitationRetry handles dialog
          │       └─ Retry after user completes elicitation
          │
          └─ Other error → TelemetrySafeError wrapper
                          → progress: { status: 'failed' }
```

### Result Content Transformation

```typescript
function transformResultContent(
  content: MCP content blocks,
  serverName: string
): ContentBlockParam[] {
  // text → { type: 'text', text: content.text }
  // image → persist blob to disk → { type: 'text', text: '[image saved to path]' }
  // audio → persist blob to disk → { type: 'text', text: '[audio saved to path]' }
  // resource → embed URI and mimeType
}
```

### Progress Events

During tool execution, progress events are emitted:

```typescript
type MCPProgress = {
  type: 'mcp_tool_started' | 'mcp_tool_completed' | 'mcp_tool_failed'
  serverName: string
  toolName: string
  elapsedTimeMs?: number
}
```

---

## Resource System

**Source**: `src/services/mcp/client.ts` — `fetchResourcesForClient()`, `src/tools/ListMcpResourcesTool/`, `src/tools/ReadMcpResourceTool/`

### Resource Discovery

```typescript
// RPC call
client.request({ method: 'resources/list' }, ListResourcesResultSchema)

// Result type
export type ServerResource = Resource & { server: string }
// Resource = { uri: string, name: string, description?: string, mimeType?: string }
```

### Resource Tools (auto-added)

When any connected server has `capabilities.resources`, two tools are automatically added to the tool pool:

**`ListMcpResourcesTool`** — lists all resources across all connected servers:
```
Input: {} (no arguments)
Output: List of resources with server names, URIs, descriptions
```

**`ReadMcpResourceTool`** — reads a specific resource:
```
Input: { uri: string, server_name: string }
Output: Resource content (text or binary blob)
```

These tools are added **once** regardless of how many servers have resources.

---

## Prompt/Command System

**Source**: `src/services/mcp/client.ts` — `fetchCommandsForClient()`

MCP servers can expose **prompts** (reusable prompt templates). In Claude Code, these become slash commands:

```
mcp__github__summarize_pr → /mcp__github__summarize_pr [args]
```

### Prompt Fetching

```typescript
// RPC call
client.request({ method: 'prompts/list' }, ListPromptsResultSchema)

// Each prompt converted to Command:
{
  type: 'prompt',
  name: `mcp__${normalizedServer}__${prompt.name}`,
  isMcp: true,
  loadedFrom: 'mcp',
  argNames: Object.keys(prompt.arguments || {}),

  async getPromptForCommand(args, context) {
    const argsArray = parseArguments(args)  // shell-quote parsing
    const namedArgs = zipObject(argNames, argsArray)

    const result = await connectedClient.getPrompt({
      name: prompt.name,
      arguments: namedArgs,
    })

    return result.messages
      .flatMap(msg => transformResultContent(msg.content, serverName))
  }
}
```

### Instructions Field

If a connected MCP server provides an `instructions` field in its `InitializeResult`, it is appended to Claude's system prompt for that session. This is the server's way of providing permanent context.

---

## MCP Skills (Feature-Gated)

**Source**: `src/skills/mcpSkillBuilders.ts`

When `feature('MCP_SKILLS')` is enabled, MCP servers can expose **skills** (structured SKILL.md-formatted prompts) in addition to plain prompts.

### Cycle-Breaking Registry Pattern

MCP skills have a circular dependency problem:

```
client.ts → mcpSkills.ts → loadSkillsDir.ts → client.ts
```

This is solved with a write-once registry:

```typescript
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b  // Called once at loadSkillsDir.ts init
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) throw new Error('MCP skill builders not registered')
  return builders
}
```

`loadSkillsDir.ts` registers the builders at module initialization (eagerly from `commands.ts`), breaking the cycle without dynamic imports.

---

## Batch Connection Orchestration

**Source**: `src/services/mcp/client.ts` — `getMcpToolsCommandsAndResources()`

At session startup, all configured MCP servers are connected in parallel with smart concurrency limits:

```typescript
// Separate local (stdio/sdk) from remote (sse/http/ws/claudeai)
const localServers = servers.filter(s => ['stdio', 'sdk'].includes(s.type))
const remoteServers = servers.filter(s => !['stdio', 'sdk'].includes(s.type))

// Connect with pMap (dynamic slot allocation, faster than fixed batches)
await pMap(localServers, connectAndFetch, {
  concurrency: getMcpServerConnectionBatchSize(),  // default 3
})

await pMap(remoteServers, connectAndFetch, {
  concurrency: getRemoteMcpServerConnectionBatchSize(),  // default 20
})
```

### Skip Conditions

Servers are skipped (not connected) if:
1. **Disabled**: `isMcpServerDisabled(name)` — turned off in settings
2. **Auth-cached**: Recently returned 401 (15-min cache) → return `needs-auth` + McpAuthTool

### onConnectionAttempt Callback

Called per-server as it completes (success or failure), enabling incremental UI updates as servers connect.

### Telemetry

```typescript
{
  totalServers,
  stdioCount,
  sseCount,
  httpCount,
  sseIdeCount,
  wsIdeCount,
}
// Per-failure: logEvent('tengu_mcp_server_connection_failed', { ...stats, transportType, connectionDurationMs })
```

---

## Caching Architecture

**Source**: `src/services/mcp/client.ts`

Three-tier cache:

### Tier 1: Connection Cache

```typescript
// Memoized by: `${serverName}-${jsonStringify(serverConfig)}`
const connectToServer = memoize(connectToServerImpl, getServerCacheKey)

// Invalidated on:
// - client.onclose event
// - Explicit clearServerCache(name, config)
// - Reconnect (clearServerCache called before reconnect)
```

### Tier 2: Tools/Resources/Prompts Cache (LRU)

```typescript
const MCP_FETCH_CACHE_SIZE = 20  // Per-cache LRU size

const fetchToolsForClient = memoizeWithLRU(fetchImpl, c => c.name, MCP_FETCH_CACHE_SIZE)
const fetchResourcesForClient = memoizeWithLRU(fetchImpl, c => c.name, MCP_FETCH_CACHE_SIZE)
const fetchCommandsForClient = memoizeWithLRU(fetchImpl, c => c.name, MCP_FETCH_CACHE_SIZE)
```

Invalidated when `clearServerCache()` is called (on connection close or reconnect).

### Tier 3: Auth Cache

```typescript
// Location: ~/.claude/mcp-needs-auth-cache.json
// Format: Record<serverId, { timestamp: number }>
// TTL: 15 minutes (MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000)
// Serialized write chain prevents concurrent write races
```

---

## Reconnection & Error Recovery

**Source**: `src/services/mcp/client.ts` — `reconnectMcpServerImpl()`

```typescript
export async function reconnectMcpServerImpl(
  name: string,
  config: ScopedMcpServerConfig,
): Promise<{ client, tools, commands, resources }> {
  clearKeychainCache()           // Force fresh credential read
  await clearServerCache(name, config)  // Evict stale connection + fetch caches

  const client = await connectToServer(name, config)  // Fresh connection

  const [tools, commands, skills, resources] = await Promise.all([
    fetchToolsForClient(client),
    fetchCommandsForClient(client),
    feature('MCP_SKILLS') ? fetchMcpSkillsForClient!(client) : [],
    client.capabilities?.resources ? fetchResourcesForClient(client) : [],
  ])

  return { client, tools: [...tools], commands: [...commands, ...skills], resources }
}
```

### Session Expiry Handling

When a tool call receives HTTP 404 + JSON-RPC error `-32001`:
1. Throw `McpSessionExpiredError`
2. Caller retries tool call **once** with a fresh connection
3. If retry also fails → propagate error

### Terminal Error Detection

These errors trigger connection close (not retry):
- `ECONNRESET` — connection reset by peer
- `ETIMEDOUT` — network timeout
- `EPIPE` — broken pipe (subprocess died)
- `EHOSTUNREACH` — no route to host
- `ECONNREFUSED` — server not listening
- `ENOTFOUND` — DNS resolution failed
- `ENETUNREACH` — network unreachable
- `ECONNABORTED` — connection aborted
- `EADDRNOTAVAIL` — address not available

---

## OAuth & Authentication

**Source**: `src/services/mcp/auth.ts`, `src/constants/oauth.ts`

### Claude.ai OAuth Scopes

```typescript
const CLAUDE_AI_OAUTH_SCOPES = [
  'user:profile',
  'user:inference',
  'user:sessions:claude_code',
  'user:mcp_servers',   // MCP-specific scope
  'user:file_upload',
]
```

### MCP Proxy URL

```typescript
MCP_PROXY_URL: 'https://mcp-proxy.anthropic.com'
MCP_PROXY_PATH: '/v1/mcp/{server_id}'
```

Claude.ai connectors route through this proxy — the connector ID maps to the `{server_id}`.

### Claude.ai Proxy Fetch Wrapper

```typescript
function createClaudeAiProxyFetch(innerFetch): FetchLike {
  return async (url, init) => {
    const doRequest = async () => {
      await checkAndRefreshOAuthTokenIfNeeded()
      const tokens = getClaudeAIOAuthTokens()
      headers.set('Authorization', `Bearer ${tokens.accessToken}`)
      return { response: await innerFetch(url, init), sentToken: tokens.accessToken }
    }

    const { response, sentToken } = await doRequest()
    if (response.status !== 401) return response

    // Single retry if token was refreshed
    const tokenChanged = await handleOAuth401Error(sentToken).catch(() => false)
    if (!tokenChanged) return response  // Genuine 401, not stale token

    return (await doRequest()).response  // One more attempt
  }
}
```

### OAuth Error Normalization

Non-standard OAuth errors are normalized to RFC-standard codes:

| Provider error | Normalized to |
|---------------|--------------|
| Slack: `invalid_refresh_token` | `invalid_grant` |
| Slack: `expired_refresh_token` | `invalid_grant` |
| Slack: `token_expired` | `invalid_grant` |

### OAuth Failure Reasons (Telemetry)

```typescript
type OAuthFailureReason =
  | 'metadata_discovery_failed'
  | 'no_client_info'
  | 'no_tokens_returned'
  | 'invalid_grant'
  | 'transient_retries_exhausted'
  | 'request_failed'
  | 'cancelled'
  | 'timeout'
  | 'provider_denied'
  | 'state_mismatch'
  | 'port_unavailable'
  | 'sdk_auth_failed'
  | 'token_exchange_failed'
  | 'unknown'
```

### Cross-App Access (XAA / SEP-990)

When `oauth.xaa: true` is set, `performCrossAppAccess()` is used instead of standard OAuth. This allows a server to authenticate using the user's existing claude.ai session (OIDC IdP discovery).

### MCP Client Metadata URL (CIMD / SEP-991)

```typescript
export const MCP_CLIENT_METADATA_URL =
  'https://claude.ai/oauth/claude-code-client-metadata'
```

Some OAuth providers use this URL to discover Claude Code's OAuth client metadata.

---

## McpAuthTool — OAuth in Practice

**Source**: `src/tools/McpAuthTool/McpAuthTool.ts`

When a server connection returns 401/403, instead of failing silently, Claude Code injects a pseudo-tool into the tool pool:

```typescript
export function createMcpAuthTool(
  serverName: string,
  config: ScopedMcpServerConfig,
): Tool<InputSchema, McpAuthOutput>

export type McpAuthOutput = {
  status: 'auth_url' | 'unsupported' | 'error'
  message: string
  authUrl?: string
}
```

### Auth Flow

```
Model calls mcp__github__create_issue
          │
          ↓
Server returns 401
          │
          ↓
Set auth cache (15-min TTL)
Set server state → 'needs-auth'
Inject McpAuthTool for server
          │
Model sees McpAuthTool in tool list
Model calls McpAuthTool
          │
          ↓
performMCPOAuthFlow({ skipBrowserOpen: true })
Returns { status: 'auth_url', authUrl: '...' }
          │
Model shows URL to user
User completes OAuth in browser
          │
          ↓ (background)
clearAuthCache(serverName)
reconnectMcpServerImpl(name, config)
          │
          ↓
Replace mcp__server__* tools in appState.mcp.tools
McpAuthTool removed from pool
Real tools available for use
```

### Supported vs Unsupported Auth

| Transport | OAuth support | Fallback |
|-----------|--------------|---------|
| `sse` | Yes — full OAuth | — |
| `http` | Yes — full OAuth | — |
| `claudeai-proxy` | Redirects to /mcp UI | — |
| `stdio` | No | Manual `/mcp` command |
| `ws` | No | Manual `/mcp` command |

---

## Elicitation System

**Source**: `src/services/mcp/elicitationHandler.ts`

Elicitation allows MCP servers to **request user input** during a tool call — not just returning data, but asking a question mid-execution.

### Protocol

```
MCP server sends ElicitRequest (JSON-RPC method: 'elicitation/create')
          │
          ↓
Claude Code receives via client.setRequestHandler(ElicitRequestSchema, ...)
          │
          ↓
1. Run elicitation hooks first (programmatic responses possible)
2. Queue request in AppState.elicitation
3. Show dialog to user (form input or URL open)
4. Wait for user response
          │
          ↓
Return ElicitResult to MCP server
          │
Server continues tool execution with user's input
```

### Elicitation in ToolUseContext

```typescript
// ToolUseContext provides:
handleElicitation?: (
  serverName: string,
  params: ElicitRequestURLParams,
  signal: AbortSignal,
) => Promise<ElicitResult>
```

### MCP Error Code Trigger

MCP error code `-32042` from a tool call triggers the elicitation callback, allowing servers to request input mid-tool-call.

### Completion Notification

```typescript
// Server sends when elicitation is done server-side:
ElicitationCompleteNotificationSchema
// Sets completed: true in AppState.elicitation queue
// Drives UI reactivity
```

---

## Plugin System & MCP Integration

**Source**: `src/plugins/`, `src/utils/plugins/`, `src/types/plugin.ts`

Plugins can bundle their own MCP servers. This is how marketplace plugins like Slack, GitHub, or custom integrations package themselves.

### Plugin Manifest (plugin.json)

```typescript
type PluginManifest = {
  name: string           // kebab-case, no spaces
  version?: string       // semver
  description?: string
  author?: { name, email?, url? }
  homepage?: string
  repository?: string
  license?: string       // SPDX identifier
  keywords?: string[]

  // References to content (paths or inline)
  commands?: string | string[] | { [name: string]: CommandMetadata }
  agents?: string | string[]
  skills?: string | string[]
  hooks?: RelativeJSONPath | HooksSchema | (...)[]

  // MCP server definitions (HIGHEST PRIORITY in manifest)
  mcpServers?: RelativeJSONPath | McpbPath | Record<string, McpServerConfig> | (...)[]

  // LSP server definitions
  lspServers?: RelativeJSONPath | Record<string, LspServerConfig> | (...)[]

  // User-configurable fields (for MCPB channels)
  userConfig?: UserConfigOption[]

  // Assistant-mode channels
  channels?: Channel[]
}
```

### Plugin MCP Server Loading

```typescript
// src/utils/plugins/mcpPluginIntegration.ts
export async function loadPluginMcpServers(
  plugin: LoadedPlugin,
  errors: PluginError[],
): Promise<Record<string, McpServerConfig> | undefined>
```

Sources checked in priority order:
1. `.mcp.json` file at plugin root
2. `manifest.mcpServers` (JSON file, MCPB bundle, or inline object)
3. `hooks/hooks.json` for hook configs

### Environment Variable Substitution in Plugin MCP Configs

```typescript
// Special variables available in plugin MCP server configs:
${CLAUDE_PLUGIN_ROOT}     → absolute path to plugin directory
${CLAUDE_PLUGIN_DATA}     → plugin data directory (~/.claude/plugin-data/{name})
${user_config.KEY}        → value saved for user config field KEY
${VAR}                    → standard environment variable
```

### Plugin Server Naming

Plugin-provided servers are namespaced:

```
"plugin:{pluginName}:{serverName}"
scope: 'dynamic'
pluginSource: "pluginName@marketplace"
```

### Deduplication

When a user has manually configured the same server that a plugin also provides:

```typescript
// Drops plugin server if it matches a manual server's signature:
// stdio: 'stdio:${jsonStringify(cmdArray)}'
// url:   'url:${unwrapCcrProxyUrl(url)}'
dedupPluginMcpServers(pluginServers, manualServers)

// Drops claude.ai connector if same server already manually configured:
dedupClaudeAiMcpServers(connectors, manualServers)
```

### Built-in Plugin Registration

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>  // MCP servers bundled with plugin
  isAvailable?: () => boolean
  defaultEnabled?: boolean
}

export function registerBuiltinPlugin(definition: BuiltinPluginDefinition): void
```

---

## MCPB / DXT Bundle Format

**Source**: `src/utils/plugins/schemas.ts`, `src/utils/plugins/mcpPluginIntegration.ts`

MCPB (MCP Bundle) and DXT are package formats for distributing MCP servers as self-contained bundles.

```typescript
// Valid MCPB paths:
McpbPath = z.union([
  RelativePath().endsWith('.mcpb' | '.dxt'),   // Local file
  z.string().url().endsWith('.mcpb' | '.dxt')  // Remote URL (downloaded at install)
])
```

### Bundle Handling

1. Download remote `.mcpb`/`.dxt` URL (if not local)
2. Extract archive contents
3. Parse DXT manifest format
4. Convert to standard `McpServerConfig`
5. Apply user configuration substitutions

### Channel / User Config

Plugins can declare configurable fields shown to users before first use:

```typescript
type UserConfigOption = {
  type: 'string' | 'number' | 'boolean' | 'directory' | 'file'
  title: string
  description: string
  required?: boolean
  default?: any
  secret?: boolean   // Stored in keychain, not settings file
}
```

Unconfigured required fields → server stays in `needs-config` state.

---

## Permission & Enterprise Controls

**Source**: `src/utils/settings/types.ts`

### Server Allow/Deny Lists (Enterprise)

```typescript
// Enterprise allowlist (if undefined = all allowed; empty array = none allowed)
allowedMcpServers?: AllowedMcpServerEntry[]

// Enterprise denylist (takes precedence over allowlist)
deniedMcpServers?: DeniedMcpServerEntry[]

type AllowedMcpServerEntry = OneOf<{
  serverName: string        // Match by name
  serverCommand: string[]   // Match by command array (stdio)
  serverUrl: string         // Match by URL pattern (supports wildcards)
}>
```

### Project Server Approval

```typescript
// From settings:
enableAllProjectMcpServers?: boolean  // Auto-approve all .mcp.json servers
enabledMcpjsonServers?: string[]      // Individually approved server names
disabledMcpjsonServers?: string[]     // Individually rejected server names

// Auto-approved if:
// - dangerouslyBypassPermissions mode is active, OR
// - Non-interactive session AND projectSettings.enableAllProjectMcpServers = true
```

### Tool-Level Permission Rules

MCP tools participate in the same permission rule system as built-in tools:

```
mcp__server1                  # All tools from server1
mcp__server1__tool_name       # Specific tool
mcp__server1__*               # Wildcard for all server1 tools
```

---

## Configuration Files & Scopes

**Source**: `src/services/mcp/config.ts`

```
Priority (later = higher):
1. /etc/claude-code/managed-mcp.json          ← Enterprise admin (highest authority)
2. ~/.claude/settings.json → mcpServers       ← User global
3. ~/.mcp.json                                ← User MCP-only config
4. <project>/.mcp.json                        ← Project servers (requires approval)
5. <project>/.claude/settings.json → mcpServers ← Project settings
6. Plugin-provided servers (scope: 'dynamic') ← Runtime
7. Claude.ai connector catalog                ← Fetched from claude.ai API
```

### .mcp.json Format

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "uvx",
      "args": ["mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp"]
    },
    "my-api": {
      "type": "http",
      "url": "https://my-server.example.com/mcp",
      "oauth": {
        "clientId": "my-client-id"
      }
    }
  }
}
```

### Config Hashing

```typescript
function hashMcpConfig(config: ScopedMcpServerConfig): string {
  return sha256(stableJsonStringify(config)).slice(0, 16)
}
// Used to detect config changes on /reload-plugins command
```

### Atomic Config Write

```typescript
// Prevents partial writes:
// 1. Write to temp file
// 2. fsync (flush to disk)
// 3. Atomic rename → target path
// 4. Preserve file permissions
async function writeMcpjsonFile(path, content): Promise<void>
```

---

## Official MCP Registry

**Source**: `src/services/mcp/officialRegistry.ts`

```typescript
// Fire-and-forget at startup
export async function prefetchOfficialMcpUrls(): Promise<void>
// Fetches: https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial
// Normalizes URLs: strip query params, trailing slashes
// Stores in in-memory Set: officialUrls

export function isOfficialMcpUrl(normalizedUrl: string): boolean
// Used for analytics: distinguish first-party vs third-party server usage
```

### Trusted Marketplace Names

```typescript
const ALLOWED_OFFICIAL_MARKETPLACE_NAMES = new Set([
  'claude-code-marketplace',
  'claude-code-plugins',
  'claude-plugins-official',
  'anthropic-marketplace',
  'anthropic-plugins',
  'agent-skills',
  'life-sciences',
  'knowledge-work-plugins',
])

const OFFICIAL_GITHUB_ORG = 'anthropics'
```

Validation rules:
- Blocks **homograph attacks** (non-ASCII characters in names)
- Blocks **impersonation** patterns: `*-official-claude`, `*claude-official*`, etc.
- Reserved names must originate from `github.com/anthropics/`
- Supports both HTTPS and SSH git URLs

Auto-update behavior:
- Official marketplaces default to `autoUpdate: true`
- Exception: `knowledge-work-plugins` defaults to `false`

---

## Claude Code as MCP Server

**Source**: `src/entrypoints/mcp.ts`, `src/server/`

Claude Code can itself act as an MCP server, exposing its tools to other clients.

### MCP Entrypoint (stdio server mode)

```typescript
// src/entrypoints/mcp.ts
// Launched when: claude --mcp-server
// Protocol: stdio (stdin/stdout)
// Exposes Claude Code's own tools as MCP tools to calling client

const server = new Server(
  { name: 'claude-code', version: MACRO.VERSION },
  { capabilities: { tools: {} } }
)

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: getTools(permissionContext).map(toMcpToolSchema)
}))

server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
  // Execute tool and return result
})

const transport = new StdioServerTransport()
await server.connect(transport)
```

### Direct-Connect Server (SDK integration)

```typescript
// src/server/createDirectConnectSession.ts
// Creates HTTP session
POST ${serverUrl}/sessions
Body: { cwd, dangerously_skip_permissions? }
Response: { session_id, ws_url, work_dir }
```

### Direct-Connect Session Manager

```typescript
// src/server/directConnectManager.ts
// DirectConnectSessionManager — manages WebSocket sessions

// Message types handled:
'control_request' subtype 'can_use_tool' → onPermissionRequest callback
'control_response', 'keep_alive', 'streamlined_*' → filtered/ignored
Other SDK messages → forwarded to onMessage callback

// Sending messages:
// User message:
{ type: 'user', message: { role, content }, parent_tool_use_id, session_id }

// Permission response:
{ type: 'control_response', response: { subtype, request_id, response } }

// Interrupt:
{ type: 'control_request', request_id, request: { subtype: 'interrupt' } }
```

---

## MCPTool — The Template Tool

**Source**: `src/tools/MCPTool/MCPTool.ts`

`MCPTool` is a template — a base tool definition whose key methods (`name`, `call`, `description`) are overridden at runtime by `fetchToolsForClient()` for each MCP tool:

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  isOpenWorld() { return false },  // Overridden per-tool from annotations
  name: 'mcp',                     // Overridden to mcp__server__tool
  maxResultSizeChars: 100_000,

  // inputSchema: passthrough z.object({}).passthrough() — JSON Schema used instead
  // outputSchema: z.string()

  async description() { return DESCRIPTION },  // Overridden
  async prompt() { return PROMPT },            // Overridden

  async call() { return { data: '' } },        // Overridden with actual RPC call

  async checkPermissions(): Promise<PermissionResult> {
    return { behavior: 'passthrough' }         // Deferred to security classifier
  },

  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return { tool_use_id: toolUseID, type: 'tool_result', content }
  },

  // Renders progress spinner with server name + tool name
  renderToolUseProgressMessage,
})
```

### Key Difference from Built-in Tools

- Uses `inputJSONSchema` (JSON Schema) instead of Zod `inputSchema` — MCP servers define their own schemas
- `checkPermissions` returns `passthrough` — security check deferred to classifier
- No `isReadOnly`/`isConcurrencySafe` by default — derived from `annotations.readOnlyHint`

---

## Timeouts Reference

| Operation | Default | Override |
|-----------|---------|---------|
| Server connection | 30,000 ms | `MCP_TIMEOUT` env var |
| Individual RPC request | 60,000 ms | (hardcoded) |
| Tool call execution | ~100,000,000 ms (27.8h) | `MCP_TOOL_TIMEOUT` env var |
| Auth cache TTL | 900,000 ms (15 min) | (hardcoded) |
| OAuth request | 30,000 ms | (hardcoded) |
| Stdio SIGINT → SIGTERM | 100 ms | (hardcoded) |
| Stdio SIGTERM → SIGKILL | 400 ms | (hardcoded) |

---

## Full Connection Flow Diagram

```
Session startup
      │
      ▼
Load MCP configs from:
  ~/.claude/settings.json   (user)
  .mcp.json                 (project)
  plugin manifests          (dynamic)
  claude.ai connectors      (remote)
  managed-mcp.json          (enterprise)
      │
      ▼
For each server:
  ├─ disabled? → DisabledMCPServer, skip
  ├─ auth-cached (15-min)? → NeedsAuthMCPServer + McpAuthTool, skip
  └─ connect:

      pMap concurrency:
      local (stdio/sdk): 3 simultaneous
      remote (sse/http/ws): 20 simultaneous
             │
             ▼
      connectToServer(name, config)
             │
        ┌────┴──────────────────────────────────────┐
        │ Select transport                           │
        ├─ stdio → StdioClientTransport (subprocess) │
        ├─ sse   → SSEClientTransport               │
        ├─ http  → StreamableHTTPClientTransport     │
        ├─ ws    → WebSocketTransport               │
        ├─ sdk   → SdkControlClientTransport        │
        └─ claudeai-proxy → Proxy fetch wrapper     │
             │
             ▼
      client.connect(transport)   ← 30s timeout
             │
        ┌────┴──────────────────┐
        │ 401/403               │ Success
        ▼                       ▼
   NeedsAuthMCPServer    ConnectedMCPServer
   McpAuthTool injected  capabilities extracted
   auth cache 15-min     instructions captured
                                │
                                ▼
                    Parallel fetch:
                    ├─ tools/list     → Tool[] (LRU cache)
                    ├─ resources/list → ServerResource[] (LRU cache)
                    └─ prompts/list   → Command[] (LRU cache)
                                │
                                ▼
                    Assemble tool pool:
                    mcp__{server}__{tool} per tool
                    ListMcpResourcesTool (once, if resources)
                    ReadMcpResourceTool  (once, if resources)
                    McpAuthTool         (per server needing auth)
                                │
                                ▼
                    Added to appState.mcp.tools
                    Available to model via SkillTool/tool list

──────────────────────────────────────────────────────

Model calls mcp__github__create_pull_request
      │
      ▼
ensureConnectedClient() → cached or reconnect
      │
      ▼
JSON-RPC: tools/call { name: "create_pull_request", arguments: {...} }
      │
      ├─ Success → transform content → return to model
      ├─ 404 + -32001 → McpSessionExpiredError → retry once with fresh conn
      ├─ -32042 → elicitation → user dialog → retry
      └─ Other → TelemetrySafeError
```

---

## Security Research Notes

1. **`headersHelper` field**: SSE/HTTP/WS configs accept a `headersHelper` field — a shell command whose stdout is parsed as JSON headers injected into all requests. This executes arbitrary shell commands at connection time, before any tool permission check.

2. **`${VAR}` env expansion in stdio configs**: Environment variables in `command` and `env` fields of stdio configs are expanded using the process environment. A malicious `.mcp.json` committed to a repo could inject `ANTHROPIC_API_KEY` or other sensitive env vars into the subprocess.

3. **Tool name normalization collision**: Server names with spaces/hyphens are normalized (`-` → `_`). Two servers named `my-server` and `my_server` produce identical tool name prefixes — one overwrites the other silently in `uniqBy`.

4. **`instructions` field injection**: The `instructions` field from `InitializeResult` is injected into Claude's system prompt verbatim. A malicious MCP server can inject arbitrary system prompt content — a direct prompt injection vector.

5. **`isOpenWorld: false` on all MCP tools**: All MCP tools return `isOpenWorld: false` by default. This may affect how the security classifier scores them — potentially too permissive.

6. **`checkPermissions: passthrough`**: MCP tools return `{ behavior: 'passthrough' }` from `checkPermissions`. Permission enforcement is deferred to an async security classifier. If the classifier is disabled or misconfigured, MCP tools may run without any permission check.

7. **15-minute auth cache blind window**: After a successful OAuth flow, the auth cache is cleared and the server reconnects. However, during the 15-minute window where a 401 is cached, the server is skipped entirely — even if the auth was fixed by the user manually.

8. **MCPB/DXT remote download**: Plugin manifests can reference remote `.mcpb`/`.dxt` URLs. These are downloaded and executed as MCP servers. A compromised package URL (CDN hijack, typosquatting) becomes a code execution vector.

9. **Plugin `${user_config.KEY}` substitution**: User-provided config values are interpolated into MCP server command arguments without any sanitization. If a user config field accepts attacker-controlled input, it could inject arguments into the subprocess command.

10. **`enableAllProjectMcpServers: true` in non-interactive mode**: When running non-interactively with this setting, ALL servers from `.mcp.json` are auto-approved without any user confirmation — a `.mcp.json` committed to a malicious repo runs its servers automatically.
