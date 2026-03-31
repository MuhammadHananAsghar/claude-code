# Claude Code Tools System — Deep Dive

> Based on source analysis of the leaked npm source map extraction (March 31, 2026).
> All type definitions, constants, and logic come directly from `src/Tool.ts`, `src/tools.ts`, `src/tools/`, `src/services/tools/`, `src/utils/permissions/`, and related files.

---

## Table of Contents

1. [Overview](#overview)
2. [Tool Interface Contract](#tool-interface-contract)
3. [buildTool Factory](#buildtool-factory)
4. [ToolUseContext — The Execution Backbone](#tooluse-context--the-execution-backbone)
5. [Tool Registration & Assembly](#tool-registration--assembly)
6. [Complete Tool Inventory (45 tools)](#complete-tool-inventory-45-tools)
7. [Tool Execution Pipeline](#tool-execution-pipeline)
8. [Concurrency Model: Serial vs Parallel](#concurrency-model-serial-vs-parallel)
9. [Streaming Tool Executor](#streaming-tool-executor)
10. [Input Validation (Zod Schemas)](#input-validation-zod-schemas)
11. [Permission System Architecture](#permission-system-architecture)
12. [Permission Rules & Matching](#permission-rules--matching)
13. [Permission Modes](#permission-modes)
14. [BashTool — The Most Complex Tool](#bashtool--the-most-complex-tool)
15. [Bash Security Layers](#bash-security-layers)
16. [Read-Only Validation](#read-only-validation)
17. [Sandbox System](#sandbox-system)
18. [Pre/Post Tool Hooks](#prepost-tool-hooks)
19. [Progress Reporting](#progress-reporting)
20. [Tool Result Handling](#tool-result-handling)
21. [MCP Tool Integration](#mcp-tool-integration)
22. [Tool Rendering & UI](#tool-rendering--ui)
23. [Telemetry & Tracing](#telemetry--tracing)
24. [Full Execution Flow Diagram](#full-execution-flow-diagram)
25. [Security Research Notes](#security-research-notes)

---

## Overview

Tools in Claude Code are the **execution primitives** — they allow the model to actually do things: read files, run commands, search the web, edit code, spawn agents. Every action the model takes that touches the real world goes through a tool.

Tools are not plugins or scripts. They are **TypeScript objects** conforming to a strict `Tool<Input, Output, Progress>` interface, registered at startup, filtered by permission context, and invoked by the model via the Anthropic API's tool-use protocol.

Key architecture facts:
- **45 built-in tools** (plus dynamic MCP tools)
- **3 execution paths**: serial, parallel (up to 10 concurrent), streaming
- **7-layer permission system** (schema → validation → hooks → rules → dialog → classifier → sandbox)
- **`buildTool()`** factory provides safe defaults — tools only override what they need
- Tools have first-class **React rendering** — each tool controls its own terminal UI

---

## Tool Interface Contract

**Source**: `src/Tool.ts`

Every tool implements this interface (most fields provided by `buildTool` defaults):

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // Identity
  name: string
  aliases?: string[]
  searchHint?: string          // Keyword hint for ToolSearchTool
  isMcp?: boolean
  isLsp?: boolean
  shouldDefer?: boolean        // Lazy-loaded (not included in cold start)
  alwaysLoad?: boolean         // Always include, even in simple mode
  mcpInfo?: { serverName: string; toolName: string }

  // Schema
  inputSchema: Input           // Zod schema — used for validation + API tool definition
  outputSchema?: z.ZodType<unknown>
  inputJSONSchema?: ToolInputJSONSchema  // MCP tools provide JSON Schema instead of Zod
  strict?: boolean             // Enforce strict Zod parsing (no extra fields)
  maxResultSizeChars: number   // Output truncation limit

  // Core execution
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  // Behavior classification
  isEnabled(): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  isConcurrencySafe(input: z.infer<Input>): boolean
  interruptBehavior?(): 'cancel' | 'block'
  requiresUserInteraction?(): boolean
  isSearchOrReadCommand?(input: z.infer<Input>): { isSearch: boolean; isRead: boolean; isList?: boolean }
  isOpenWorld?(input: z.infer<Input>): boolean

  // Permission & validation
  checkPermissions(input: z.infer<Input>, context: ToolUseContext): Promise<PermissionResult>
  validateInput?(input: z.infer<Input>, context: ToolUseContext): Promise<ValidationResult>
  preparePermissionMatcher?(input: z.infer<Input>): Promise<(pattern: string) => boolean>

  // UI rendering (React/Ink)
  renderToolUseMessage(input, options): React.ReactNode
  renderToolUseProgressMessage?(progress[], options): React.ReactNode
  renderToolUseQueuedMessage?(): React.ReactNode
  renderToolResultMessage?(content, progress[], options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderGroupedToolUse?(toolUses[], options): React.ReactNode | null
  renderToolUseTag?(input): React.ReactNode

  // Descriptive text
  description(input: z.infer<Input>, options): Promise<string>
  prompt(options): Promise<string>              // System prompt contribution
  userFacingName(input): string
  userFacingNameBackgroundColor?(input): keyof Theme | undefined
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null

  // Result mapping
  mapToolResultToToolResultBlockParam(content: Output, toolUseID: string): ToolResultBlockParam
  extractSearchText?(out: Output): string
  isResultTruncated?(output: Output): boolean

  // Misc
  getPath?(input: z.infer<Input>): string
  toAutoClassifierInput(input: z.infer<Input>): unknown
  inputsEquivalent?(a: z.infer<Input>, b: z.infer<Input>): boolean
  backfillObservableInput?(input: Record<string, unknown>): void
}
```

### ToolResult Type

```typescript
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

The `contextModifier` field lets tools mutate the execution context for subsequent tool calls (used by `EnterPlanModeTool`, `EnterWorktreeTool`, etc.).

---

## buildTool Factory

**Source**: `src/Tool.ts`

All tools use `buildTool()` which applies safe defaults so tools only implement what they override:

```typescript
const TOOL_DEFAULTS = {
  isEnabled:          () => true,
  isConcurrencySafe:  (_input?) => false,       // Safe default: serial
  isReadOnly:         (_input?) => false,        // Safe default: not read-only
  isDestructive:      (_input?) => false,
  checkPermissions:   (input, _ctx?) =>          // Safe default: allow all
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',
  userFacingName:     (_input?) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,               // Tool-specific overrides win
  } as BuiltTool<D>
}
```

### Minimal Tool Example

```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  searchHint: 'do something useful',
  maxResultSizeChars: 100_000,
  strict: true,

  inputSchema: z.object({
    target: z.string().describe('What to operate on'),
  }),

  description: async ({ target }) => `Operate on ${target}`,
  prompt: async () => 'Use MyTool when you need to operate on things.',
  userFacingName: ({ target }) => target ?? 'MyTool',

  isReadOnly: () => true,
  isConcurrencySafe: () => true,   // Can run in parallel with other tools

  async call(input, context, canUseTool, parentMessage, onProgress) {
    return { data: { result: 'done' } }
  },
})
```

---

## ToolUseContext — The Execution Backbone

**Source**: `src/Tool.ts`

Every tool call receives a `ToolUseContext` — a rich object providing access to the entire session state:

```typescript
export type ToolUseContext = {
  // Configuration
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    verbose: boolean
    debug: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    querySource?: QuerySource
    refreshTools?: () => Tools
  }

  // Lifecycle
  abortController: AbortController

  // State
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  setAppStateForTasks?: (f: (prev: AppState) => AppState) => void
  messages: Message[]

  // File caching
  readFileState: FileStateCache

  // UI
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif: Notification) => void
  appendSystemMessage?: (msg: SystemMessage) => void
  sendOSNotification?: (opts: { message: string; notificationType: string }) => void

  // Progress tracking
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  setResponseLength: (f: (prev: number) => number) => void
  setStreamMode?: (mode: SpinnerMode) => void

  // Permissions
  toolDecisions?: Map<string, { source: string; decision: 'accept' | 'reject'; timestamp: number }>
  localDenialTracking?: DenialTrackingState
  requireCanUseTool?: boolean

  // Agent context
  agentId?: AgentId
  agentType?: string

  // Memory/skills integration
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  dynamicSkillDirTriggers?: Set<string>
  discoveredSkillNames?: Set<string>

  // File history
  updateFileHistoryState: (updater) => void
  updateAttributionState: (updater) => void

  // Limits
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }

  // Misc
  setConversationId?: (id: UUID) => void
  userModified?: boolean
  queryTracking?: QueryChainTracking
  preserveToolUseResults?: boolean
  contentReplacementState?: ContentReplacementState
  renderedSystemPrompt?: SystemPrompt
  criticalSystemReminder_EXPERIMENTAL?: string
  toolUseId?: string
  pushApiMetricsEntry?: (ttftMs: number) => void
  handleElicitation?: (serverName, params, signal) => Promise<ElicitResult>
  requestPrompt?: (sourceName, toolInputSummary?) => (request) => Promise<PromptResponse>
  onCompactProgress?: (event: CompactProgressEvent) => void
  setSDKStatus?: (status: SDKStatus) => void
  openMessageSelector?: () => void
}
```

---

## Tool Registration & Assembly

**Source**: `src/tools.ts`

### getAllBaseTools()

Returns the complete list of all built-in tools:

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    // + conditional tools based on feature flags and environment
  ]
}
```

### getTools(permissionContext)

Filters the base tool list by permissions and mode:

```typescript
export const getTools = (permissionContext: ToolPermissionContext): Tools => {
  // Simple mode — return minimal tool set
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    return [BashTool, FileReadTool, FileEditTool]
  }

  const tools = getAllBaseTools().filter(t => !specialTools.has(t.name))
  let allowedTools = filterToolsByDenyRules(tools, permissionContext)

  // Hide tools replaced by REPL mode
  if (isReplModeEnabled()) {
    allowedTools = allowedTools.filter(t => !REPL_ONLY_TOOLS.has(t.name))
  }

  // Run isEnabled() check (can gate on feature flags, settings, OS)
  return allowedTools.filter(t => t.isEnabled())
}
```

### assembleToolPool(permissionContext, mcpTools)

Combines built-in and MCP tools, deduplicates, sorts for cache stability:

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

---

## Complete Tool Inventory (45 tools)

**Source**: `src/tools/` directory

| Tool | Name | Read-Only | Concurrent | Purpose |
|------|------|-----------|------------|---------|
| `AgentTool` | Agent | No | No | Spawn sub-agents with isolated contexts |
| `AskUserQuestionTool` | AskUserQuestion | No | No | Interactive user prompts |
| `BashTool` | Bash | Conditional | Conditional | Shell command execution |
| `BriefTool` | Brief | Yes | Yes | Display content briefly |
| `ConfigTool` | Config | No | No | Read/write settings |
| `EnterPlanModeTool` | EnterPlanMode | Yes | Yes | Switch to plan-only mode |
| `EnterWorktreeTool` | EnterWorktree | No | No | Enter git worktree isolation |
| `ExitPlanModeV2Tool` | ExitPlanMode | No | No | Exit plan mode, write plan to disk |
| `ExitWorktreeTool` | ExitWorktree | No | No | Exit worktree |
| `FileEditTool` | Edit | No | No | Edit files with string replacement |
| `FileReadTool` | Read | Yes | Yes | Read file contents |
| `FileWriteTool` | Write | No | No | Write entire files |
| `GlobTool` | Glob | Yes | Yes | Pattern-based file matching |
| `GrepTool` | Grep | Yes | Yes | Regex content search via ripgrep |
| `LSPTool` | LSP | Yes | Yes | Language Server Protocol queries |
| `ListMcpResourcesTool` | ListMcpResources | Yes | Yes | List MCP server resources |
| `MCPTool` | mcp__* | Varies | Varies | MCP server tool invocation |
| `McpAuthTool` | McpAuth | No | No | MCP OAuth authentication |
| `NotebookEditTool` | NotebookEdit | No | No | Edit Jupyter notebook cells |
| `PowerShellTool` | PowerShell | Conditional | No | Windows PowerShell execution |
| `REPLTool` | REPL | No | No | Persistent Node.js REPL VM |
| `ReadMcpResourceTool` | ReadMcpResource | Yes | Yes | Read MCP server resources |
| `RemoteTriggerTool` | RemoteTrigger | No | No | Trigger remote agent execution |
| `ScheduleCronTool` | ScheduleCron | No | No | Schedule recurring cron jobs |
| `SendMessageTool` | SendMessage | Conditional | No | Send messages to team agents |
| `SkillTool` | Skill | No | No | Invoke skills (slash commands) |
| `SleepTool` | Sleep | Yes | Yes | Wait for duration |
| `SyntheticOutputTool` | SyntheticOutput | No | No | Structured output injection |
| `TaskCreateTool` | Task | No | No | Create background tasks |
| `TaskGetTool` | TaskGet | Yes | Yes | Get task details |
| `TaskListTool` | TaskList | Yes | Yes | List all tasks |
| `TaskOutputTool` | TaskOutput | Yes | Yes | Read task output |
| `TaskStopTool` | TaskStop | No | No | Kill a running task |
| `TaskUpdateTool` | TaskUpdate | No | No | Update task properties |
| `TeamCreateTool` | TeamCreate | No | No | Create agent teams |
| `TeamDeleteTool` | TeamDelete | No | No | Delete agent teams |
| `TodoWriteTool` | TodoWrite | No | No | Manage todo list |
| `ToolSearchTool` | ToolSearch | Yes | Yes | Search for deferred tool schemas |
| `WebFetchTool` | WebFetch | Yes | Yes | HTTP GET with markdown conversion |
| `WebSearchTool` | WebSearch | Yes | Yes | Web search via Claude's search API |

---

## Tool Execution Pipeline

**Source**: `src/services/tools/toolExecution.ts`

When the model calls a tool (via a `tool_use` block in the API response), the execution pipeline runs:

```
API response contains tool_use block
        │
        ▼
1. FIND TOOL
   findToolByName(context.options.tools, toolUse.name)
   → checks aliases too
   → if not found: yield error message, return
        │
        ▼
2. PARSE & VALIDATE INPUT (Zod)
   tool.inputSchema.safeParse(toolUse.input)
   → on failure: yield schema error message, return
        │
        ▼
3. TOOL-SPECIFIC VALIDATION
   tool.validateInput?(parsedInput, context)
   → e.g. BashTool blocks sleep patterns here
   → ValidationResult { result: boolean, message?, errorCode? }
        │
        ▼
4. BACKFILL OBSERVABLE INPUT
   tool.backfillObservableInput?(input)
   → fills in derived fields for telemetry
        │
        ▼
5. PRE-TOOL-USE HOOKS
   runPreToolUseHooks(context, tool, input, ...)
   → Each hook can:
     - Yield a message
     - Make a permission decision (hookPermissionResult)
     - Modify input (hookUpdatedInput)
     - Prevent continuation
     - Provide a stop reason
     - Add additional context
        │
        ▼
6. PERMISSION CHECK
   canUseTool(tool, processedInput, context, assistantMessage, toolUseID)
   → behavior: 'allow' → proceed
   → behavior: 'deny' → yield denial message, return
   → behavior: 'ask' → show permission UI, wait for user
        │
        ▼
7. TOOL EXECUTION
   tool.call(callInput, context, canUseTool, parentMessage, onProgress)
   → onProgress callback yields progress messages during execution
   → returns ToolResult<Output>
        │
        ▼
8. POST-TOOL-USE HOOKS
   runPostToolUseHooks(context, tool, input, result, ...)
   → Can modify result, add messages
        │
        ▼
9. RESULT MAPPING
   tool.mapToolResultToToolResultBlockParam(result.data, toolUseID)
   → Converts to Anthropic API ToolResultBlockParam
   → Large results persisted to disk if > threshold
        │
        ▼
10. CONTEXT MODIFICATION
    result.contextModifier?.(context)
    → Applied after execution for non-concurrent tools
    → Updates ToolUseContext for subsequent calls

11. YIELD TOOL RESULT
    Sent back as user message to continue conversation
```

---

## Concurrency Model: Serial vs Parallel

**Source**: `src/services/tools/toolOrchestration.ts`

When the model outputs multiple tool calls in a single response, Claude Code decides whether to run them concurrently or serially.

### Partitioning

```typescript
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[] {
  return toolUseMessages.reduce((acc: Batch[], toolUse) => {
    const tool = findToolByName(...)
    const isConcurrencySafe = tool?.isConcurrencySafe(parsedInput.data) ?? false

    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1]!.blocks.push(toolUse)  // Merge into concurrent batch
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] })  // New batch
    }
    return acc
  }, [])
}
```

### Execution

**Concurrent batch** (all `isConcurrencySafe === true`):
```typescript
// Max concurrency: CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY env var, default 10
async function* runToolsConcurrently(toolUseMessages, ...) {
  yield* all(
    toolUseMessages.map(async function* (toolUse) {
      setInProgressToolUseIDs(prev => new Set(prev).add(toolUse.id))
      yield* runToolUse(toolUse, ...)
      markToolUseAsComplete(context, toolUse.id)
    }),
    getMaxToolUseConcurrency(),
  )
}
```

**Serial batch** (any tool has `isConcurrencySafe === false`):
```typescript
async function* runToolsSerially(toolUseMessages, ...) {
  let currentContext = toolUseContext
  for (const toolUse of toolUseMessages) {
    for await (const update of runToolUse(toolUse, ...currentContext)) {
      if (update.newContext) currentContext = update.newContext  // Context modifiers applied
      yield update
    }
  }
}
```

### Context Modifiers

Only serial tools can modify the shared execution context (e.g., changing permission mode, switching worktree). Concurrent tools cannot — they get a snapshot of the context at batch start.

### Error Cascading

- **Bash errors** → abort sibling tools via `siblingAbortController`
- **Other tool errors** → do NOT cascade; siblings continue

---

## Streaming Tool Executor

**Source**: `src/services/tools/StreamingToolExecutor.ts` (~530 lines)

For streaming API responses, tools arrive as the model generates them — not all at once. The `StreamingToolExecutor` manages execution as tools stream in:

```typescript
export class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  private siblingAbortController: AbortController
  private hasErrored = false

  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    const isConcurrencySafe = toolDefinition?.isConcurrencySafe(parsedInput.data) ?? false
    this.tools.push({
      id: block.id,
      block,
      status: 'queued',
      isConcurrencySafe,
      pendingProgress: [],
    })
    void this.processQueue()  // Non-blocking
  }

  private canExecuteTool(tool: TrackedTool): boolean {
    // Concurrent-safe: can execute if no non-concurrent tool is running
    // Non-concurrent: must wait for all in-progress tools to finish
  }

  private async executeTool(tool: TrackedTool): Promise<void> {
    tool.status = 'executing'
    for await (const update of runToolUse(tool.block, ...)) {
      if (update.isProgress) {
        tool.pendingProgress.push(update)  // Progress emitted immediately
      } else {
        tool.result = update
      }
    }
    tool.status = 'completed'
  }
}
```

**Guarantee**: Results are always yielded in the order tools were received, regardless of completion order.

---

## Input Validation (Zod Schemas)

**Source**: `src/tools/*/` — each tool defines its own schema

All tool inputs are defined as Zod schemas which serve dual purpose:
1. **Validation**: `tool.inputSchema.safeParse(rawInput)` before execution
2. **API schema**: Converted to JSON Schema for the Anthropic tools API

### Example Schemas

**BashTool** (full input schema):
```typescript
const fullInputSchema = lazySchema(() => z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: semanticNumber(z.number().optional())
    .describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
  description: z.string().optional()
    .describe('Clear, concise description of what this command does'),
  run_in_background: semanticBoolean(z.boolean().optional())
    .describe('Set to true to run this command in the background'),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional())
    .describe('Set this to true to dangerously override sandbox mode'),
  _simulatedSedEdit: z.object({    // INTERNAL ONLY — omitted from model-facing schema
    filePath: z.string(),
    newContent: z.string()
  }).optional()
}))

// Model-facing schema hides internal field:
const inputSchema = lazySchema(() => fullInputSchema().omit({ _simulatedSedEdit: true }))
```

**GrepTool**:
```typescript
z.object({
  pattern: z.string(),
  path: z.string().optional(),
  glob: z.string().optional(),
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  '-A': z.number().optional(),    // Lines after
  '-B': z.number().optional(),    // Lines before
  '-C': z.number().optional(),    // Context lines
  '-n': z.boolean().optional(),   // Show line numbers
  '-i': z.boolean().optional(),   // Case insensitive
  type: z.string().optional(),    // ripgrep --type
  head_limit: z.number().optional(),
  offset: z.number().optional(),
  multiline: z.boolean().optional(),
})
```

**TodoWriteTool**:
```typescript
z.object({
  todos: z.array(TodoSchema)  // Full replacement of todo list
})
```

### `semanticBoolean` / `semanticNumber`

Custom Zod wrappers that accept both typed values and string representations from the model:

```typescript
// semanticBoolean accepts "true"/"false" strings in addition to booleans
// semanticNumber accepts numeric strings in addition to numbers
// These handle model output imprecision
```

---

## Permission System Architecture

**Source**: `src/types/permissions.ts`, `src/utils/permissions/`

The permission system has 8 layers executed in order:

```
Layer 1: Zod schema validation
Layer 2: tool.validateInput() — tool-specific structural checks
Layer 3: Pre-tool hooks — hooks can make permission decisions
Layer 4: canUseTool() — the main permission decision function
   ├── Layer 4a: Permission rules (allow/deny/ask from all sources)
   ├── Layer 4b: Tool.checkPermissions() — tool's own logic
   └── Layer 4c: Classifier check (async, ML-based)
Layer 5: Permission dialog — user prompt if behavior='ask'
Layer 6: Post-tool hooks
Layer 7: Sandbox enforcement (OS-level)
Layer 8: Result validation
```

### Core Permission Types

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask'

export type PermissionResult<Input = unknown> =
  | { behavior: 'allow'; updatedInput?: Input; decisionReason? }
  | { behavior: 'ask'; message: string; updatedInput?: Input; suggestions?: PermissionUpdate[]; pendingClassifierCheck?: ... }
  | { behavior: 'deny'; message: string; decisionReason }
  | { behavior: 'passthrough'; ... }

export type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'other'; reason: string }
```

---

## Permission Rules & Matching

**Source**: `src/utils/permissions/permissions.ts`

### Rule Structure

```typescript
export type PermissionRuleValue = {
  toolName: string        // e.g., "Bash", "Read", "WebFetch"
  ruleContent?: string    // e.g., "git commit", "domain:github.com", "~/projects/**"
}

export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}
```

### Rule Sources (priority order, last wins)

```typescript
export type PermissionRuleSource =
  | 'policySettings'   // Enterprise/org admin policies (highest authority)
  | 'userSettings'     // ~/.claude/settings.json
  | 'projectSettings'  // .claude/settings.json
  | 'localSettings'    // .claude/.claude-local-settings.json
  | 'flagSettings'     // --permission-* CLI flags
  | 'cliArg'           // --allow-tools argument
  | 'command'          // Agent/programmatic grant
  | 'session'          // Runtime session grant (lowest authority)
```

### Rule Pattern Syntax

```
Bash                          # All Bash tool uses
Bash(git log)                 # Exact: "git log" only
Bash(git:*)                   # Prefix wildcard: any "git ..." command
Bash(git log:*)               # Prefix wildcard: "git log ..." commands
Bash(*:--dry-run)             # Suffix pattern
WebFetch(domain:github.com)   # Domain-specific URL fetch
Read(//absolute/path/**)      # Absolute path pattern
Read(~/project/**)            # Home-relative path
Edit(./src/**)                # Project-relative path
mcp__server1                  # All tools from MCP server "server1"
mcp__server1__tool_name       # Specific MCP tool
```

### Rule Matching

```typescript
function toolMatchesRule(
  tool: Pick<Tool, 'name' | 'mcpInfo'>,
  rule: PermissionRule,
): boolean {
  if (rule.ruleValue.ruleContent !== undefined) return false  // Content rules handled separately
  const nameForCheck = getToolNameForPermissionCheck(tool)
  if (rule.ruleValue.toolName === nameForCheck) return true   // Direct match

  // MCP server-level match: "mcp__server" matches "mcp__server__tool"
  const ruleInfo = mcpInfoFromString(rule.ruleValue.toolName)
  const toolInfo = mcpInfoFromString(nameForCheck)
  return ruleInfo?.serverName === toolInfo?.serverName &&
         (ruleInfo?.toolName === undefined || ruleInfo?.toolName === '*')
}
```

---

## Permission Modes

**Source**: `src/types/permissions.ts`

```typescript
export type PermissionMode =
  | 'default'           // Prompt for unrecognized tools
  | 'dontAsk'           // Auto-deny if not in explicit allow list
  | 'acceptEdits'       // Auto-allow file edits, prompt for others
  | 'bypassPermissions' // Allow everything (requires explicit user opt-in)
  | 'plan'              // Plan mode: read-only tools only
  | 'auto'              // ML classifier-based auto-approval
  | 'bubble'            // Internal: hierarchical permission evaluation

export const PERMISSION_MODES = [
  'default', 'dontAsk', 'acceptEdits', 'bypassPermissions', 'plan', 'auto'
]
```

### ToolPermissionContext

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource  // Per-source allow rules
  alwaysDenyRules: ToolPermissionRulesBySource   // Per-source deny rules
  alwaysAskRules: ToolPermissionRulesBySource    // Per-source ask rules
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource  // Rules removed by hooks
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

---

## BashTool — The Most Complex Tool

**Source**: `src/tools/BashTool/BashTool.tsx` (~1,000 lines)

BashTool is the most complex tool. Its `isReadOnly()` is dynamic — it inspects the actual command:

```typescript
isReadOnly(input) {
  const compoundCommandHasCd = commandHasAnyCd(input.command)
  const result = checkReadOnlyConstraints(input, compoundCommandHasCd)
  return result.behavior === 'allow'
},

isConcurrencySafe(input) {
  return this.isReadOnly?.(input) ?? false
},
```

### Output Schema

```typescript
const outputSchema = z.object({
  stdout: z.string(),
  stderr: z.string(),
  rawOutputPath: z.string().optional(),
  interrupted: z.boolean(),
  isImage: z.boolean().optional(),
  backgroundTaskId: z.string().optional(),
  backgroundedByUser: z.boolean().optional(),
  assistantAutoBackgrounded: z.boolean().optional(),
  dangerouslyDisableSandbox: z.boolean().optional(),
  returnCodeInterpretation: z.string().optional(),
  noOutputExpected: z.boolean().optional(),
  structuredContent: z.array(z.any()).optional(),
  persistedOutputPath: z.string().optional(),    // Path to large output on disk
  persistedOutputSize: z.number().optional(),
})
```

### Execution

```typescript
async call(input, toolUseContext, _canUseTool, parentMessage, onProgress) {
  // Handle simulated sed edits (internal optimization)
  if (input._simulatedSedEdit) return applySedEdit(...)

  const commandGenerator = runShellCommand({
    input,
    abortController,
    setAppState,
    setToolJSX,
    preventCwdChanges: !!toolUseContext.agentId,  // Sub-agents can't cd
    isMainThread: !toolUseContext.agentId,
    toolUseId,
    agentId,
  })

  // Stream progress
  for await (const progress of commandGenerator) {
    onProgress?.({
      toolUseID: `bash-progress-${progressCounter++}`,
      data: {
        type: 'bash_progress',
        output: progress.output,
        fullOutput: progress.fullOutput,
        elapsedTimeSeconds: progress.elapsedTimeSeconds,
        totalLines: progress.totalLines,
        totalBytes: progress.totalBytes,
        taskId: progress.taskId,
        timeoutMs: progress.timeoutMs,
      },
    })
  }

  // Handle large outputs — persist to disk
  if (result.outputFilePath) {
    // Copy to ~/.claude/tool-results/{id}
    persistedOutputPath = ...
    persistedOutputSize = ...
  }
}
```

---

## Bash Security Layers

**Source**: `src/tools/BashTool/bashPermissions.ts` (2,621 lines), `src/tools/BashTool/bashSecurity.ts` (2,592 lines)

BashTool has the most elaborate security system of any tool. Seven distinct security checks:

### 1. Command Prefix Extraction

```typescript
export function getSimpleCommandPrefix(command: string): string | null
// "git commit -m 'fix'" → "git commit"
// Returns null if: unsafe env vars, token looks suspicious
```

### 2. Security Pattern Detection (bashSecurity.ts)

Blocks or warns on dangerous patterns:

| Pattern | Example | Action |
|---------|---------|--------|
| Command substitution | `$(cmd)`, \`cmd\`, `<(cmd)` | deny |
| Variable injection | `$IFS`, `/proc/self/environ` | deny |
| Process substitution | `>(cmd)`, `=(cmd)`, `~[]` | deny |
| Zsh dangerous builtins | `zmodload`, `emulate`, `zpty`, `ztcp` | deny |
| Heredoc in substitution | `<<EOF` in `$(...)` | deny |
| Brace expansion injection | `{cmd,other}` abuse | deny |
| Obfuscated base commands | `sed 's/a/b/g'` to hide real command | deny |
| ANSI-C quoting exploits | `$'...'` abuse | deny |
| Nested shell | `bash -c "..."`, `sh -c` | ask |

### 3. Quote Extraction Analysis

```typescript
type QuoteExtraction = {
  withDoubleQuotes: string        // Unquoted singles removed, double quotes kept
  fullyUnquoted: string           // All quotes stripped
  unquotedKeepQuoteChars: string  // Quotes stripped, delimiters preserved
}

function extractQuotedContent(command: string, isJq = false): QuoteExtraction
// Handles bash quoting semantics precisely
// Tracks: '', "", $'', nested/escaped quotes
```

### 4. Safe Wrapper Stripping

```typescript
export function stripSafeWrappers(command: string): string
// Strips: sudo, env, xargs -I {}, timeout, strace, ltrace
// Prevents "sudo bash" bypassing "Bash(bash:*)" deny rule
```

### 5. Tree-sitter AST Parsing

```typescript
export async function parseForSecurity(command: string): Promise<ParseForSecurityResult>
// Uses tree-sitter to parse bash AST
// Handles compound commands, heredocs, special syntax
// Returns: { kind: 'simple' | 'compound', commands: ... }
```

### 6. Compound Command Handling

```typescript
export const MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50
export const MAX_SUGGESTED_RULES_FOR_COMPOUND = 5

// Each subcommand in "cmd1 && cmd2 | cmd3" is checked independently
// Collects and deduplicates suggested permission rules
// Returns 'ask' if any subcommand returns 'ask'
// Returns 'deny' if any subcommand returns 'deny'
```

### 7. Permission Rule Matching

```typescript
export function matchWildcardPattern(rulePattern: string, command: string): boolean
// "git *" matches "git commit -m foo"
// Supports exact, prefix, suffix, wildcard patterns
```

---

## Read-Only Validation

**Source**: `src/tools/BashTool/readOnlyValidation.ts` (~1,990 lines)

Determines if a bash command is truly read-only (and thus can run concurrently).

The file contains a large allowlist of safe commands with precise flag configurations:

```typescript
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: {
      '-I': '{}',
      '-E': 'EOF',
      '-n': 'number',
      '-P': 'number',
      '-0': 'none',
      // NOTE: -i/-e intentionally removed (GNU getopt optional-arg confusion)
    },
  },
  git: { /* all GIT_READ_ONLY_COMMANDS */ },
  find: { /* safe flags only, -exec excluded */ },
  'fd': { /* -x/--exec excluded */ },
  // ... 40+ tools
}
```

**Safe commands list includes**: cat, head, tail, wc, grep, rg, ls, du, df, stat, file, type, which, whereis, locate, echo, printf, date, uname, hostname, env, printenv, id, whoami, groups, git (read ops), curl (GET), wget (no-execute), tar (list), zip/unzip (list), openssl (read), jq (no system()), python -c (limited), node -e (limited), etc.

**Excluded from safe**: `find -exec`, `fd -x`, `xargs -i` (optional arg exploit risk), `jq` with file args or system(), eval, exec, etc.

---

## Sandbox System

**Source**: `src/utils/sandbox/sandbox-adapter.ts`, `src/entrypoints/sandboxTypes.ts`

The sandbox system provides OS-level isolation for bash commands.

### SandboxManager

```typescript
export class SandboxManager extends BaseSandboxManager {
  static isSandboxingEnabled(): boolean
  static areUnsandboxedCommandsAllowed(): boolean
  static isAutoAllowBashIfSandboxedEnabled(): boolean
  static getSandboxConfig(): SandboxRuntimeConfig
}
```

### Configuration Sources (priority order)

```
1. policySettings.sandbox.*   ← Enterprise admin (highest)
2. projectSettings.sandbox.*  ← .claude/settings.json
3. localSettings.sandbox.*    ← .claude/.claude-local-settings.json
4. userSettings.sandbox.*     ← ~/.claude/settings.json (lowest)
```

### dangerouslyDisableSandbox

```typescript
// BashTool input field:
dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional())
  .describe('Set this to true to dangerously override sandbox mode...')

// Usage in shouldUseSandbox.ts:
if (input.dangerouslyDisableSandbox &&
    SandboxManager.areUnsandboxedCommandsAllowed()) {
  return false  // Bypass sandbox — must be explicitly permitted by policy
}

// Permission decision reason:
{ type: 'sandboxOverride', reason: 'dangerouslyDisableSandbox' }
```

`areUnsandboxedCommandsAllowed()` is a policy gate — `dangerouslyDisableSandbox: true` in the input is **not sufficient** on its own. The sandbox policy must permit unsandboxed execution.

### Sandbox Restrictions → Permission Rules

The sandbox configuration is translated into permission rules:

```typescript
// Network domains → WebFetch(domain:X) permission rules
// File read paths → Read(//path/**) rules
// File write paths → Edit(//path/**) rules
// Filesystem restrictions → nested_memory/plugins allowlists

// Path pattern translation:
// //path/to/**   → absolute from root
// /path/to/**    → relative to settings dir
// ~/path/**      → home directory
// ./path         → relative to cwd
```

---

## Pre/Post Tool Hooks

**Source**: `src/hooks/`, `src/services/tools/toolExecution.ts`

Hooks execute before and after every tool call. They can:

### PreToolUse Hook Outcomes

```typescript
type PreToolUseHookResult =
  | { type: 'message'; message: Message }
  | { type: 'hookPermissionResult'; permissionResult: PermissionResult }
  | { type: 'hookUpdatedInput'; updatedInput: unknown }        // Modify tool input
  | { type: 'preventContinuation' }                           // Stop tool, continue loop
  | { type: 'stopReason'; reason: string }                    // Stop entire session
  | { type: 'additionalContext'; context: string }            // Add context
  | { type: 'stop' }                                          // Immediate stop
```

### PostToolUse Hook Outcomes

```typescript
type PostToolUseHookResult =
  | { type: 'message'; message: Message }
  | { type: 'updatedOutput'; output: ToolResult }             // Modify tool result
  | { type: 'preventContinuation' }
  | { type: 'stopReason'; reason: string }
  | { type: 'stop' }
```

### Hook Configuration

Hooks are configured in `~/.claude/settings.json` or `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "echo 'About to run bash'" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          { "type": "command", "command": "prettier --write $CLAUDE_TOOL_INPUT_FILE_PATH" }
        ]
      }
    ]
  }
}
```

---

## Progress Reporting

**Source**: `src/Tool.ts`, `src/types/tools.ts`

Tools report streaming progress via the `onProgress` callback:

```typescript
export type ToolCallProgress<P extends ToolProgressData> = (
  progress: ToolProgress<P>,
) => void

export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}

export type ToolProgressData =
  | BashProgress
  | WebSearchProgress
  | AgentToolProgress
  | REPLToolProgress
  | SkillToolProgress
  | MCPProgress
  | TaskOutputProgress
  | { type: unknown }  // Extensible

export type BashProgress = {
  type: 'bash_progress'
  output: string
  fullOutput: string
  elapsedTimeSeconds: number
  totalLines: number
  totalBytes: number
  taskId?: string
  timeoutMs?: number
}

export type WebSearchProgress = {
  type: 'query_update' | 'search_results_received'
  query?: string
  resultCount?: number
}
```

Progress messages are yielded **immediately** by the StreamingToolExecutor — they don't wait for tool completion.

---

## Tool Result Handling

**Source**: `src/Tool.ts`, `src/services/tools/toolExecution.ts`

### Result Mapping

Each tool implements `mapToolResultToToolResultBlockParam()` to convert its output to the Anthropic API format:

```typescript
// Standard text result:
mapToolResultToToolResultBlockParam(content, toolUseID) {
  return {
    type: 'tool_result',
    tool_use_id: toolUseID,
    content: typeof content === 'string' ? content : JSON.stringify(content),
  }
}
```

### Large Output Persistence

When tool output exceeds `maxResultSizeChars`, it is persisted to disk:

```
~/.claude/tool-results/{task-id}
```

The model receives a truncated preview + path reference. This prevents context window overflow from large command outputs.

### maxResultSizeChars by Tool

| Tool | Limit |
|------|-------|
| BashTool | 30,000 chars |
| FileReadTool | 100,000 chars |
| GrepTool | 100,000 chars |
| WebFetchTool | 100,000 chars |
| MCPTool | 100,000 chars |

---

## MCP Tool Integration

**Source**: `src/tools/MCPTool/MCPTool.ts`, `src/services/mcp/`

MCP (Model Context Protocol) tools are dynamically registered at runtime from connected MCP servers.

### MCPTool Template

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  isOpenWorld() { return false },
  name: 'mcp',                          // Overridden per-tool in mcpClient.ts
  maxResultSizeChars: 100_000,
  async call() { return { data: '' } }, // Overridden per-tool
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return { tool_use_id: toolUseID, type: 'tool_result', content }
  },
})
```

### MCP Tool Naming

MCP tools are prefixed: `mcp__{serverName}__{toolName}`

Examples:
- `mcp__github__create_pull_request`
- `mcp__context7__query-docs`
- `mcp__playwright__browser_screenshot`

### MCP Tool Permissions

MCP tools go through the same permission system as built-in tools. Rules can target:
- `mcp__server1` → all tools from server1
- `mcp__server1__tool_name` → specific tool
- `mcp__server1__*` → wildcard for server1 tools

### JSON Schema vs Zod

MCP tools provide JSON Schema (not Zod) for their inputs:

```typescript
inputJSONSchema?: ToolInputJSONSchema  // For MCP tools
inputSchema: Input                     // For built-in tools (Zod)
```

---

## Tool Rendering & UI

**Source**: Each tool's `render*` methods, `src/components/`

Every tool is responsible for its own terminal UI rendering via React/Ink:

```typescript
// Show what tool is about to do:
renderToolUseMessage(input, { theme, verbose, commands }): React.ReactNode

// Show streaming progress:
renderToolUseProgressMessage(progress[], options): React.ReactNode

// Show queued indicator:
renderToolUseQueuedMessage?(): React.ReactNode

// Show final result:
renderToolResultMessage?(content, progress[], options): React.ReactNode

// Show error:
renderToolUseErrorMessage?(result, options): React.ReactNode

// Show when user rejected:
renderToolUseRejectedMessage?(input, options): React.ReactNode

// Group multiple uses (e.g., Read tool shows file list):
renderGroupedToolUse?(toolUses[], options): React.ReactNode | null
```

This architecture means each tool has complete control over how it appears in the terminal — the BashTool shows a spinner with live output, the FileEditTool shows a diff, the WebSearchTool shows search queries as they execute.

---

## Telemetry & Tracing

**Source**: `src/services/tools/toolExecution.ts`, `src/services/analytics/`

### OpenTelemetry Spans

Every tool execution is traced:

```typescript
startToolSpan(toolName, input)
startToolExecutionSpan(toolName)
startToolBlockedOnUserSpan(toolName)
endToolSpan()
endToolExecutionSpan()
endToolBlockedOnUserSpan()
addToolContentEvent(toolName, contentSize)
```

### Analytics Events (Datadog/firstParty)

```typescript
logEvent('tengu_tool_use_diff_computed', {
  isEditTool: true,
  durationMs: Date.now() - startTime,
  hasDiff: !!diff,
})

logEvent('tengu_tool_search_outcome', {
  query: queryValue,
  queryType: 'select' | 'keyword',
  matches: string[],
})

logEvent('tengu_exit_plan_mode_called_outside_plan', {
  model: mainLoopModel,
  mode: toolPermissionContext.mode,
  hasExitedPlanModeInSession: boolean,
})
```

---

## Full Execution Flow Diagram

```
Anthropic API streams response
        │
        ▼
content_block_start { type: "tool_use", name: "Bash", id: "tu_123" }
        │
        ▼
StreamingToolExecutor.addTool(block)
  └── processQueue() → canExecuteTool()?
        │
   ┌────┴─────────────────────────────────┐
   │ isConcurrencySafe = true             │ isConcurrencySafe = false
   │                                      │
   ▼                                      ▼
 Execute in parallel with           Wait for all in-progress
 other concurrent-safe tools        tools to complete first
   │                                      │
   └────────────────┬─────────────────────┘
                    │
                    ▼
         runToolUse(toolUseBlock, ...)
                    │
                    ▼
         1. findToolByName()
                    │
                    ▼
         2. inputSchema.safeParse(input)
            → fail: yield SchemaError message
                    │
                    ▼
         3. tool.validateInput(parsedInput)
            → fail: yield ValidationError message
                    │
                    ▼
         4. tool.backfillObservableInput(input)
            → fills telemetry fields
                    │
                    ▼
         5. runPreToolUseHooks(...)
            → hooks may: modify input, deny, stop session
                    │
                    ▼
         6. canUseTool(tool, input, context, ...)
            → check deny rules → deny immediately
            → check allow rules → allow immediately
            → tool.checkPermissions() → tool-specific logic
            → behavior: 'ask' → show permission UI
                    │
                    ▼
         7. tool.call(input, context, canUseTool, parentMessage, onProgress)
            │
            ├── onProgress() called during execution
            │   → progress messages yielded immediately
            │
            └── returns ToolResult<Output>
                    │
                    ▼
         8. runPostToolUseHooks(...)
            → hooks may: modify result, stop session
                    │
                    ▼
         9. tool.mapToolResultToToolResultBlockParam(result, toolUseID)
            → converts to Anthropic API format
            → large results persisted to disk
                    │
                    ▼
        10. yield tool_result as user message
            → loops back to API call with result
```

---

## Security Research Notes

1. **`_simulatedSedEdit` hidden field**: BashTool's full input schema includes `_simulatedSedEdit` which is omitted from the model-facing schema. This field lets internal code inject pre-computed edit results without the model knowing. Worth investigating how this internal channel can be triggered.

2. **`dangerouslyDisableSandbox` requires policy gate**: The field exists in the input schema and is documented in the system prompt, but bypassing sandbox also requires `SandboxManager.areUnsandboxedCommandsAllowed()` to return true. However, the *decision reason* is logged — could be used to audit when sandbox bypass occurs.

3. **`contextModifier` in ToolResult**: Tools can return a function that mutates the `ToolUseContext`. This is a powerful primitive — a tool that returns a malicious `contextModifier` could change permission mode, swap the tool pool, or modify working directories for all subsequent tool calls.

4. **Pre-tool hooks can modify input**: `hookUpdatedInput` allows a hook to silently rewrite the tool's input. A malicious PostToolUse hook (configured in `.claude/settings.json`) could modify results after execution.

5. **MCP tools bypass Zod validation**: MCP tools provide `inputJSONSchema` (JSON Schema) instead of a Zod schema. JSON Schema validation is less strict than Zod's runtime type checking — type coercions and edge cases differ.

6. **`alwaysLoad` tools bypass mode filtering**: Tools with `alwaysLoad: true` are included even in simplified/restricted modes. Understanding which tools have this flag reveals the minimal tool surface that always exists.

7. **Sub-agents inherit tool pool**: `AgentTool` spawns sub-agents that inherit the parent's `ToolUseContext`. The sub-agent permission rules are the parent's rules unless the agent definition specifies overrides. A skill that spawns a sub-agent effectively gets the same permissions as the parent.

8. **`requireCanUseTool: false` context field**: `ToolUseContext` has a `requireCanUseTool` field. When false, permission checks may be skipped entirely for certain internal execution paths.

9. **Bash sibling error propagation**: Only Bash errors cascade to abort sibling tools. Other tool errors (file not found, network error) don't abort siblings. This asymmetry means carefully crafted Bash failures can interrupt other concurrent operations.

10. **`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` env var**: Controls the concurrency limit for parallel tool execution (default 10). Setting this very high could create resource exhaustion scenarios.
