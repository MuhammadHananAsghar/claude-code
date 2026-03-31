# Claude Code — Source Analysis

> **This is Anthropic's real Claude Code CLI source**, extracted via an npm source map leak on **March 31, 2026**.  
> Discovered by [@Fried_rice (Chaofan Shou)](https://twitter.com/Fried_rice).  
> Analysis by [Muhammad Hanan Asghar](https://github.com/MuhammadHananAsghar).

> Static analysis only — the code cannot be compiled or run as-is.

---

## Authenticity

### Authentication Proof

| Evidence | Details |
|----------|---------|
| OAuth Client IDs | Hardcoded UUID: `9d1c250a-e61b-44d9-88ed-5944d1962f5e` (prod) |
| API Endpoints | `api.anthropic.com`, `platform.claude.com/v1/oauth/token` |
| Internal Domains | `.ant.dev` (Anthropic-only infra), `artifactory.infra.ant.dev` |
| Slack Channels | Multiple `anthropic.slack.com/archives/C0xxxxx` references |
| SDK | Uses `@anthropic-ai/sdk` with proper client initialization |

### Source Map Extraction Signature

1,679 of 1,884 `.ts` files use `.js` extensions in imports — the unmistakable fingerprint of source-map reconstruction. Normal TypeScript repos don't do this.

```typescript
// Example from the extracted code:
import { getOauthConfig } from '../constants/oauth.js'  // .js in .ts = source map extraction
```

### Feature Match — 100%

| Feature | Present |
|---------|---------|
| Tools (Read, Write, Edit, Bash, Grep, Glob, Agent) | ✅ 45 tools |
| Slash commands (/commit, /compact, /config, etc.) | ✅ 103+ commands |
| MCP server integration | ✅ |
| Remote sessions (bridge / CCR) | ✅ |
| Voice mode | ✅ |
| Vim mode | ✅ |
| Plan mode | ✅ |
| Worktree isolation | ✅ |
| Agent SDK | ✅ |

### Scale

| Metric | Value |
|--------|-------|
| Total files | 1,902 |
| Lines of TypeScript | 512,685 |
| Built-in tools | 45 |
| Slash commands | 103+ |
| MCP transport types | 6 |
| Internal feature flags | PROACTIVE, KAIROS, BRIDGE_MODE, DAEMON, VOICE_MODE |
| Missing (expected) | `package.json`, `tsconfig.json`, build config — not stored in source maps |

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [Tools System](docs/tools-system.md) | Full tool interface, execution pipeline, permission architecture |
| [Tools List](docs/tools-list.md) | All 45 tools with inputs, descriptions, read-only/concurrency flags |
| [MCP System](docs/mcp-system.md) | Model Context Protocol — transports, OAuth, elicitation, plugins |
| [Memory System](docs/memory-system.md) | 4-tier memory, CLAUDE.md hierarchy, auto-dream consolidation |
| [Skills System](docs/skills-system.md) | SKILL.md-backed slash commands, prompt expansion pipeline |
| [Slash Commands](docs/slash-commands.md) | All 103+ slash commands with availability and descriptions |
| [Plan Mode System](docs/plan-mode-system.md) | 5-phase planning workflow, agents, approval flow |
| [Remote Sessions](docs/remote-sessions-system.md) | CCR bridge, WebSocket/SSE transports, worktree provisioning |
| [API Endpoints](docs/api-endpoints.md) | All 68 HTTP/WebSocket endpoints with auth and headers |

---

## Architecture Overview

```mermaid
graph TB
    User["👤 User / claude.ai"]

    subgraph CLI["Claude Code CLI"]
        REPL["REPL / Terminal UI\n(React + Ink)"]
        CMD["Slash Commands\n103+ commands"]
        QUERY["Query Engine\n(main loop)"]
        TOOLS["Tool Executor\n45 built-in tools"]
        PERMS["Permission System\n7-layer check"]
        MEMORY["Memory System\nCLAUDE.md hierarchy"]
        MCP["MCP Client\n6 transport types"]
        PLAN["Plan Mode\n5-phase workflow"]
        BRIDGE["Remote Bridge\nCCR v1/v2"]
    end

    API["Anthropic API\napi.anthropic.com"]
    MCPSERVERS["MCP Servers\n(external)"]
    REMOTE["claude.ai\n(remote control)"]

    User --> REPL
    REPL --> CMD
    REPL --> QUERY
    QUERY --> TOOLS
    QUERY --> MEMORY
    TOOLS --> PERMS
    PERMS --> TOOLS
    TOOLS --> MCP
    MCP --> MCPSERVERS
    QUERY --> API
    QUERY --> PLAN
    BRIDGE --> REMOTE
    CLI --> BRIDGE
```

---

## Tool Execution Flow

```mermaid
flowchart TD
    A["Model returns tool_use block"] --> B["assembleToolPool()"]
    B --> C{"Tool found?"}
    C -- No --> ERR["Return error result"]
    C -- Yes --> D["validateInput (Zod schema)"]
    D --> E{"Valid?"}
    E -- No --> ERR
    E -- Yes --> F["Pre-hooks\n(PreToolUse)"]
    F --> G{"Hook outcome?"}
    G -- block --> ERR
    G -- approve --> I
    G -- continue --> H["checkPermissions()"]
    H --> I{"Permission?"}
    I -- deny --> J["Show permission dialog\n(user approval)"]
    J --> K{"User approves?"}
    K -- No --> ERR
    K -- Yes --> I2["Permission granted"]
    I -- allow --> I2
    I2 --> L["tool.call(input, context)"]
    L --> M["Post-hooks\n(PostToolUse)"]
    M --> N["mapToolResultToToolResultBlockParam()"]
    N --> O["Return to model"]

    style ERR fill:#ff6b6b,color:#fff
    style I2 fill:#51cf66,color:#fff
```

---

## Permission System (7 Layers)

```mermaid
flowchart LR
    A["Tool call"] --> B["1. Zod\nvalidation"]
    B --> C["2. validateInput()\ncustom checks"]
    C --> D["3. Pre-hooks\nPreToolUse"]
    D --> E["4. Permission rules\nallow/deny lists"]
    E --> F["5. checkPermissions()\ntool-specific logic"]
    F --> G{"6. Dialog\n(if needed)"}
    G -- approved --> H["7. Sandbox check\nSandboxManager"]
    H --> EXEC["✅ Execute"]
    G -- denied --> BLOCK["🚫 Blocked"]

    style EXEC fill:#51cf66,color:#fff
    style BLOCK fill:#ff6b6b,color:#fff
```

---

## MCP Flow

```mermaid
sequenceDiagram
    participant Claude as Claude Code CLI
    participant MCPClient as MCP Client
    participant Transport as Transport Layer
    participant Server as MCP Server

    Claude->>MCPClient: getTools() / getResources()
    MCPClient->>Transport: initialize connection
    
    alt stdio transport
        Transport->>Server: spawn subprocess
    else SSE transport
        Transport->>Server: GET /sse (EventSource)
    else HTTP transport
        Transport->>Server: POST /mcp
    end

    Server-->>Transport: capabilities response
    Transport-->>MCPClient: connected
    MCPClient-->>Claude: tool list (cached LRU-20)

    Note over Claude,Server: Tool call flow

    Claude->>MCPClient: callTool(name, input)
    MCPClient->>Transport: JSON-RPC request
    Transport->>Server: tools/call
    Server-->>Transport: result
    Transport-->>MCPClient: response
    
    alt 401 Unauthorized
        MCPClient->>Claude: trigger McpAuthTool
        Claude->>Server: OAuth flow
        Server-->>Claude: token
        Claude->>MCPClient: retry callTool
    end

    MCPClient-->>Claude: tool result
```

---

## MCP Transport Types

```mermaid
graph LR
    MCP["MCP Client"] --> T1["stdio\nsubprocess spawn"]
    MCP --> T2["SSE\nServer-Sent Events"]
    MCP --> T3["HTTP\nJSON-RPC POST"]
    MCP --> T4["WebSocket\nws:// connection"]
    MCP --> T5["SDK\nIn-process bridge"]
    MCP --> T6["claudeai-proxy\nhttps://mcp-proxy.anthropic.com"]

    T1 --> S1["Local CLI tools\ngit, filesystem, etc"]
    T2 --> S2["Remote servers\nwith SSE endpoint"]
    T3 --> S3["Stateless remote\nservers"]
    T4 --> S4["Bidirectional\nreal-time servers"]
    T5 --> S5["Claude Code as\nMCP server"]
    T6 --> S6["Official registry\nservers"]
```

---

## Memory System

```mermaid
graph TD
    subgraph Sources["Memory Sources (priority order)"]
        M1["1. Enterprise CLAUDE.md\n/etc/claude/CLAUDE.md"]
        M2["2. User CLAUDE.md\n~/.claude/CLAUDE.md"]
        M3["3. Project CLAUDE.md\n{git-root}/CLAUDE.md"]
        M4["4. Local CLAUDE.md\n.claude/CLAUDE.md"]
        M5["5. Auto memory\n~/.claude/projects/{hash}/memory/"]
        M6["6. Team memory\nGitHub-synced entries"]
        M7["7. Agent memory\n~/.claude/agents/memory/"]
    end

    subgraph MEMINDEX["MEMORY.md Index"]
        IDX["MEMORY.md\nmax 200 lines / 25KB\nAI picks up to 5 relevant files"]
    end

    subgraph Types["Memory Types"]
        T1["user — role, preferences"]
        T2["feedback — corrections, patterns"]
        T3["project — goals, decisions"]
        T4["reference — external pointers"]
    end

    M1 & M2 & M3 & M4 --> QUERY["Query Engine\n(injected into system prompt)"]
    M5 --> IDX --> QUERY
    M6 --> QUERY
    M7 --> QUERY
    IDX --> T1 & T2 & T3 & T4
```

---

## Plan Mode Flow

```mermaid
flowchart TD
    START(["User / Claude calls\nEnterPlanMode"]) --> PREP["prepareContextForPlanMode()\nSave prePlanMode\nStrip dangerous permissions"]
    PREP --> MODE["mode = 'plan'"]

    MODE --> PH1["Phase 1: Explore\nLaunch 3 Explore agents in parallel\nRead files, grep, glob"]
    PH1 --> PH2["Phase 2: Design\nLaunch 1–3 Plan agents\nWrite to plan file (.md)"]
    PH2 --> PH3["Phase 3: Review\nMain loop reviews all plans\nReconcile trade-offs"]
    PH3 --> PH4["Phase 4: Final Plan\nFormat final plan file\n~/.claude/plans/{slug}.md"]
    PH4 --> PH5["Phase 5: ExitPlanMode\nPresent plan to user"]

    PH5 --> APPROVAL{"User approves?"}
    APPROVAL -- No --> PH2
    APPROVAL -- Yes --> RESTORE["Restore prePlanMode\nRe-enable permissions"]
    RESTORE --> IMPL(["Begin implementation"])

    subgraph TeamFlow["Teammate Flow (plan_mode_required)"]
        TM["Teammate calls ExitPlanMode"] --> MAILBOX["Send plan_approval_request\nto team lead via mailbox"]
        MAILBOX --> LEAD{"Team lead\napproves?"}
        LEAD -- Yes --> IMPL
        LEAD -- No --> TM
    end

    style IMPL fill:#51cf66,color:#fff
    style MODE fill:#339af0,color:#fff
```

---

## Remote Sessions (CCR) Flow

```mermaid
sequenceDiagram
    participant User as User (claude.ai)
    participant CLI as Claude Code CLI
    participant API as Anthropic API
    participant CCR as CCR Session Ingress
    participant Transport as Transport (SSE/WS)

    Note over CLI,API: V2 Environment-less path

    CLI->>API: POST /v1/code/sessions
    API-->>CLI: { session_id }

    CLI->>API: POST /v1/code/sessions/{id}/bridge
    API-->>CLI: { worker_jwt, api_base_url }

    CLI->>CCR: POST /worker/register
    CCR-->>CLI: { worker_epoch }

    CLI->>Transport: Connect SSE stream\n/worker/events/stream
    Transport-->>CLI: connected

    loop Session active
        User->>API: Send message (claude.ai)
        API->>Transport: Stream event (inbound message)
        Transport->>CLI: onData callback
        CLI->>CLI: Execute tools, generate response
        CLI->>CCR: POST response chunks (HTTP)
        CCR->>API: Forward to claude.ai
        CLI->>CCR: POST /worker/heartbeat (every 20s)
    end

    CLI->>API: POST /v1/sessions/{id}/archive
```

---

## Remote Sessions Transport Selection

```mermaid
flowchart LR
    ENV{"Env var?"} 
    ENV -- "CLAUDE_CODE_USE_CCR_V2=1" --> SSE["SSETransport\nRead: SSE stream\nWrite: HTTP POST batches"]
    ENV -- "CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1" --> HYBRID["HybridTransport\nRead: WebSocket\nWrite: HTTP POST batches"]
    ENV -- "default" --> WS["WebSocketTransport\nRead + Write: WebSocket"]

    SSE --> CCR["CCRClient\nHeartbeat 20s\nEpoch tracking\nEvent upload queues"]
    HYBRID --> CCR
    WS --> DIRECT["Direct WS connection\nto session ingress"]
```

---

## Skills / Slash Command Resolution

```mermaid
flowchart TD
    INPUT["User types /skill-name"] --> LOOKUP["getCommands(cwd)"]

    LOOKUP --> S1["Bundled skills\n17 built-in SKILL.md files"]
    LOOKUP --> S2["User skills\n~/.claude/skills/*.md\n.claude/skills/*.md"]
    LOOKUP --> S3["Plugin skills\nfrom installed plugins"]
    LOOKUP --> S4["Built-in commands\n103+ hardcoded"]
    LOOKUP --> S5["Workflow scripts\n.claude/workflows/*.md"]

    S1 & S2 & S3 --> SKILL["SkillTool.call()"]
    S4 --> LOCALCMD["Local JS command\nor prompt expansion"]
    S5 --> WORKFLOW["WorkflowTool.call()"]

    SKILL --> EXPAND["Prompt expansion pipeline\n1. baseDir substitution\n2. $ARGUMENTS injection\n3. $CLAUDE_SKILL_DIR\n4. $CLAUDE_SESSION_ID\n5. Inline shell execution"]
    EXPAND --> MODEL["Sent to model\nas user message"]

    LOCALCMD --> MODEL
    WORKFLOW --> MODEL
```

---

## OAuth Authentication Flow

```mermaid
sequenceDiagram
    participant CLI as Claude Code CLI
    participant Browser as Browser
    participant Auth as api.anthropic.com/oauth
    participant API as Anthropic API

    CLI->>CLI: Generate PKCE code_verifier + code_challenge
    CLI->>Browser: Open authorize URL\n(client_id, scope, code_challenge)
    Browser->>Auth: User logs in + approves scopes
    Auth-->>Browser: Redirect with auth code
    Browser-->>CLI: Capture redirect (local server)

    CLI->>Auth: POST /v1/oauth/token\n{ code, code_verifier, client_id }
    Auth-->>CLI: { access_token, refresh_token, expires_in }

    CLI->>Auth: GET /api/oauth/profile
    Auth-->>CLI: { org_uuid, user_id, subscription_type }

    Note over CLI,API: Token stored in OS keychain

    loop Token expired (8h TTL)
        CLI->>Auth: POST /v1/oauth/token\n{ grant_type: refresh_token }
        Auth-->>CLI: new access_token
    end

    CLI->>API: All requests\nAuthorization: Bearer {access_token}\nx-organization-uuid: {org_uuid}
```

---

## Agent System

```mermaid
graph TD
    AGENT["Agent tool call"] --> TYPE{"subagent_type"}

    TYPE --> GP["general-purpose\nAll tools available"]
    TYPE --> EXPLORE["Explore\nRead-only tools\nGlob, Grep, Read, Bash(ro)"]
    TYPE --> PLAN["Plan\nRead-only + plan file write\nNo nested agents"]
    TYPE --> GUIDE["claude-code-guide\nContext7, WebSearch, WebFetch"]
    TYPE --> VERIFY["verification\nTest runner, linter"]
    TYPE --> STATUSLINE["statusline-setup\nRead + Edit only"]
    TYPE --> CUSTOM["Custom agents\n~/.claude/agents/*.md\nor .claude/agents/*.md"]

    EXPLORE --> FORK["Fork subprocess\nIsolated context"]
    PLAN --> FORK
    GP --> FORK

    FORK --> ISOLATED["Isolated memory\nOwn tool pool\nOwn permission context"]
    ISOLATED --> RESULT["Result returned\nto parent agent"]

    subgraph Worktree["Optional: Isolation mode"]
        WT["isolation: 'worktree'\nFresh git worktree\nClean branch"]
    end

    FORK -.-> WT
```

---

## Security Architecture

```mermaid
graph LR
    subgraph Threats["Threat Surface"]
        T1["User input"]
        T2["File system"]
        T3["Shell execution"]
        T4["MCP servers"]
        T5["Remote sessions"]
    end

    subgraph Defenses["Defense Layers"]
        D1["Zod schema validation\non all tool inputs"]
        D2["Path traversal checks\ncontainsPathTraversal()"]
        D3["tree-sitter AST\nbash security analysis"]
        D4["MCP tool annotations\nreadOnlyHint, destructiveHint"]
        D5["OAuth + Worker JWT\n+ Trusted Device Token"]
        D6["Sandbox execution\nSandboxManager"]
        D7["Permission allow/deny lists\nper-project + per-session"]
        D8["Plan mode\nread-only phase"]
    end

    T1 --> D1
    T2 --> D2
    T3 --> D3 --> D6
    T4 --> D4 --> D7
    T5 --> D5
    D7 --> D8
```

---

## Codebase Structure

```
src/
├── tools/              # 45 built-in tools (one dir per tool)
├── commands/           # 103+ slash commands
├── services/
│   └── mcp/            # MCP client, transports, auth
├── bridge/             # Remote sessions (CCR v1/v2)
├── cli/
│   └── transports/     # WebSocket, SSE, Hybrid transports
├── upstreamproxy/      # CONNECT-over-WebSocket tunnel
├── utils/
│   ├── permissions/    # Permission system
│   ├── plans.ts        # Plan file management
│   ├── worktree.ts     # Git worktree provisioning
│   └── teamMemory.ts   # Team memory sync
├── skills/             # Bundled SKILL.md files
├── types/              # Shared TypeScript types
├── components/         # React/Ink terminal UI components
├── constants/          # OAuth config, feature flags
└── entrypoints/        # CLI, MCP server, bridge daemon
```

---

## Key Internal Feature Flags

These `bun:bundle` compile-time flags gate entire subsystems:

| Flag | Unlocks |
|------|---------|
| `KAIROS` | Channels mode (Telegram, Discord, Slack bots) |
| `BRIDGE_MODE` | Remote control daemon + `/bridge` command |
| `VOICE_MODE` | Voice input/output |
| `PROACTIVE` | Proactive suggestion system |
| `DAEMON` | Background daemon process |
| `ULTRAPLAN` | Extended ultra plan mode |
| `FORK_SUBAGENT` | `/fork` conversation branching |
| `WORKFLOW_SCRIPTS` | `.claude/workflows/` script runner |
| `TRANSCRIPT_CLASSIFIER` | Auto permission mode via transcript analysis |
| `TEAMMEM` | Team memory sync |
| `TORCH` | Internal Anthropic workflow |
| `BUDDY` | Buddy agent mode |
| `UDS_INBOX` | Unix socket peer-to-peer inbox |

---

> **Disclaimer:** This repository contains source code obtained through a source map leak in Anthropic's npm package distribution. It is published here for security research and educational purposes only.
