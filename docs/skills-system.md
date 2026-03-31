# Claude Code Skills System — Deep Dive

> Based on source analysis of the leaked npm source map extraction (March 31, 2026).
> All type definitions, constants, and logic come directly from `src/skills/`, `src/tools/SkillTool/`, `src/types/command.ts`, `src/commands.ts`, and related files.

---

## Table of Contents

1. [Overview](#overview)
2. [What is a Skill?](#what-is-a-skill)
3. [Command Type System](#command-type-system)
4. [Skill File Format](#skill-file-format)
5. [Frontmatter Reference](#frontmatter-reference)
6. [Skill Sources & Loading Order](#skill-sources--loading-order)
7. [Bundled Skills](#bundled-skills)
8. [Plugin Skills & MCP Skills](#plugin-skills--mcp-skills)
9. [Command Registry & Aggregation](#command-registry--aggregation)
10. [SkillTool — How the Model Invokes Skills](#skilltool--how-the-model-invokes-skills)
11. [Slash Command Invocation (User-Facing)](#slash-command-invocation-user-facing)
12. [Execution Modes: Inline vs Fork](#execution-modes-inline-vs-fork)
13. [Prompt Expansion Pipeline](#prompt-expansion-pipeline)
14. [Argument Substitution System](#argument-substitution-system)
15. [Conditional Skills (Path-Triggered)](#conditional-skills-path-triggered)
16. [Skill Visibility Filters](#skill-visibility-filters)
17. [Context Window Budget for Skills](#context-window-budget-for-skills)
18. [Hooks in Skills](#hooks-in-skills)
19. [Permission & Isolation Model](#permission--isolation-model)
20. [Skill Caching Architecture](#skill-caching-architecture)
21. [Skillify — The Meta Skill](#skillify--the-meta-skill)
22. [Full Data Flow Diagram](#full-data-flow-diagram)
23. [Security Research Notes](#security-research-notes)

---

## Overview

Skills are Claude Code's primary extensibility mechanism. They are **prompt-based commands** — markdown files with YAML frontmatter — that expand into instructions for the model when invoked, either by the user (`/skill-name`) or by the model itself via the `SkillTool`.

Skills are fundamentally different from tools:

| | **Tools** | **Skills** |
|-|-----------|-----------|
| Format | TypeScript code | Markdown + YAML |
| Who writes them | Anthropic | Anyone (users, plugins, Anthropic for bundled) |
| What they do | Execute code, call APIs, read/write files | Expand into prompts/instructions for the model |
| Invocation | Model calls them directly | User via `/name` or model via `SkillTool` |
| Isolation | Sandboxed execution | Inline context or forked sub-agent |

---

## What is a Skill?

A skill is a `PromptCommand` — a command whose `type === 'prompt'`. When invoked, its markdown content is expanded and injected into the conversation as a user message, effectively telling the model "here are the instructions for this task."

Minimal example:

```
~/.claude/skills/
└── greet/
    └── SKILL.md
```

```markdown
---
description: Greet the user warmly
user-invocable: true
---

Please greet the user and ask how you can help them today.
```

Running `/greet` injects that prompt into the conversation.

---

## Command Type System

**Source**: `src/types/command.ts`

Everything in Claude Code's command system (skills, slash commands, interactive menus) flows through a single `Command` union type:

```typescript
export type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

| Variant | `type` field | What it is |
|---------|-------------|-----------|
| `PromptCommand` | `'prompt'` | Skills — expands to prompt content |
| `LocalCommand` | `'local'` | In-process TypeScript function |
| `LocalJSXCommand` | `'local-jsx'` | Interactive Ink/React TUI component |

### CommandBase (shared fields)

```typescript
type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  userInvocable?: boolean           // Can user type /name?
  disableModelInvocation?: boolean  // Prevent model from calling via SkillTool
  isHidden?: boolean                // Hide from /help listing
  isMcp?: boolean                   // Sourced from MCP server
  hasUserSpecifiedDescription?: boolean
  whenToUse?: string                // Detailed guidance for model on when to use
  argumentHint?: string             // e.g. "[branch-name] [message]"
  version?: string
  availability?: CommandAvailability[]
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  kind?: 'workflow'
  immediate?: boolean               // Execute immediately without confirmation
  isSensitive?: boolean
  userFacingName?: () => string
}
```

### PromptCommand (skill-specific fields)

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number             // Character count of expanded prompt
  argNames?: string[]               // Named argument list from frontmatter
  allowedTools?: string[]           // Tool allowlist for this skill
  model?: string                    // Override model for this skill
  hooks?: HooksSettings             // Register hooks on invocation
  skillRoot?: string                // Base directory (for ${CLAUDE_SKILL_DIR})
  context?: 'inline' | 'fork'      // Execution mode
  agent?: string                    // Agent type when context='fork'
  effort?: EffortValue              // Token budget hint
  paths?: string[]                  // Glob patterns for conditional activation
  getPromptForCommand(
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>   // Returns expanded prompt content
}
```

---

## Skill File Format

**Source**: `src/skills/loadSkillsDir.ts`

Skills must use the directory format with a `SKILL.md` file:

```
~/.claude/skills/
└── {skill-name}/
    ├── SKILL.md          ← required, defines the skill
    ├── helper.sh         ← optional supporting files
    ├── template.md       ← referenced via ${CLAUDE_SKILL_DIR}/template.md
    └── ...
```

A flat `.md` file is also supported (legacy, treated as `commands_DEPRECATED`):

```
~/.claude/commands/
└── my-command.md         ← legacy flat format
```

### Full SKILL.md Example

```markdown
---
name: review-pr
description: Review a GitHub pull request for quality, security, and correctness
user-invocable: true
argument-hint: "[pr-number-or-url]"
arguments: pr_url
when-to-use: >
  Use when the user wants to review a PR, check a pull request, or
  analyze changes in a branch. Works with PR numbers or full GitHub URLs.
allowed-tools:
  - Bash(gh pr view:*)
  - Bash(gh pr diff:*)
  - Read
  - Grep
model: claude-opus-4-6
effort: hard
context: fork
agent: code-reviewer
version: 1.2.0
hooks:
  PostToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: echo "tool used in skill"
paths:
  - "*.ts"
  - "src/**"
---

# PR Review

Review the pull request at: $pr_url

Focus on:
1. Code quality and correctness
2. Security vulnerabilities
3. Test coverage
4. Architecture consistency

Use `gh pr view $pr_url` to get PR details and `gh pr diff $pr_url` for changes.

Base directory for supporting files: ${CLAUDE_SKILL_DIR}
Session: ${CLAUDE_SESSION_ID}
```

---

## Frontmatter Reference

**Source**: `src/skills/loadSkillsDir.ts` — `parseSkillFrontmatterFields()`

| Key | Type | Default | Effect |
|-----|------|---------|--------|
| `name` | string | directory name | Display name for the skill |
| `description` | string | auto-generated | One-line description shown in /help and SkillTool listing |
| `user-invocable` | boolean | `true` | Whether user can invoke via `/skill-name` |
| `disable-model-invocation` | boolean | `false` | Block model from calling via SkillTool |
| `when-to-use` | string | — | Guidance for model on when to use this skill |
| `argument-hint` | string | — | UI hint, e.g. `[branch] [message]` |
| `arguments` | string | — | Space-separated named argument list for `$argname` substitution |
| `allowed-tools` | string[] | all tools | Tool allowlist for this skill's execution |
| `model` | string | current model | Override model for this skill |
| `effort` | `hard\|medium\|easy\|<number>` | — | Token budget hint for forked execution |
| `context` | `inline\|fork` | `inline` | Execution mode |
| `agent` | string | — | Agent type when `context: fork` |
| `hooks` | object | — | Hooks to register when skill runs |
| `paths` | string[] | — | Glob patterns — skill only activates when matching files are touched |
| `shell` | `inherit\|no-shell\|restrict` | `inherit` | Shell mode for inline `!`\`...\` commands |
| `version` | string | — | Skill version for tracking |

---

## Skill Sources & Loading Order

**Source**: `src/skills/loadSkillsDir.ts`, `src/commands.ts`

Skills are loaded from multiple sources and merged. Priority (later sources override earlier ones when names conflict):

```
1. Managed skills     /etc/claude-code/.claude/skills/          ← policy-enforced
2. User skills        ~/.claude/skills/                          ← personal library
3. Project skills     <project>/.claude/skills/                  ← project-specific
4. Ancestor skills    <parent-dirs-up-to-home>/.claude/skills/   ← monorepo hierarchy
5. --add-dir skills   any/dir/.claude/skills/                    ← CLI flag
6. Legacy commands    ~/.claude/commands/*.md                    ← deprecated
                      <project>/.claude/commands/*.md
```

### Source Tags

Each loaded skill gets a `source` field (for provenance tracking) and `loadedFrom`:

| `loadedFrom` | `source` | Where it came from |
|-------------|---------|-------------------|
| `'bundled'` | `'bundled'` | Shipped with Claude Code binary |
| `'skills'` | `'global_settings'\|'local_settings'` | Disk-based SKILL.md files |
| `'plugin'` | `'plugin'` | Installed plugins |
| `'managed'` | `'global_managed'` | Admin-enforced policy skills |
| `'mcp'` | `'mcp'` | MCP server-provided skills |
| `'commands_DEPRECATED'` | varies | Legacy .claude/commands/*.md |

### Loading Function

```typescript
// src/skills/loadSkillsDir.ts
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]>
)

// Internally calls:
async function loadSkillsFromSkillsDir(
  basePath: string,
  source: SettingSource,
): Promise<SkillWithPath[]>

// For legacy format:
async function loadSkillsFromCommandsDir(
  cwd: string,
): Promise<SkillWithPath[]>
```

---

## Bundled Skills

**Source**: `src/skills/bundledSkills.ts`, `src/skills/bundled/`

Bundled skills are programmatically registered at startup — their content is compiled into the binary (via Bun's bundler), not loaded from disk. They cannot be modified by users.

### Registration API

```typescript
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean        // Runtime gate (feature flags, settings)
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>   // Files extracted to ~/.claude/bundled-skills/{name}/
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}

export function registerBundledSkill(definition: BundledSkillDefinition): void
export function getBundledSkills(): Command[]
export function getBundledSkillExtractDir(skillName: string): string
// → ~/.claude/bundled-skills/{skill-name}/
```

### Bundled Skills List

From `src/skills/bundled/index.ts`:

| Skill | Name | User-Invocable | Notes |
|-------|------|---------------|-------|
| `updateConfig` | update-config | — | Modify settings.json via conversation |
| `keybindings` | keybindings-help | — | Keybinding configuration |
| `verify` | verify | Yes | Content verification |
| `debug` | debug | — | Debugging assistance |
| `loremIpsum` | lorem-ipsum | — | Placeholder text generation |
| `skillify` | skillify | Yes | Create skills from session workflows |
| `remember` | remember | Yes | Explicit memory saving |
| `simplify` | simplify | — | Code simplification |
| `batch` | batch | — | Batch operations |
| `stuck` | stuck | — | Help when stuck |
| `dream` | dream | — | Memory consolidation (KAIROS mode, feature-gated) |
| `hunter` | bughunter | — | Bug hunting (feature-gated) |
| `loop` | loop | — | Recurring skill execution (feature-gated) |
| `schedule` | schedule | — | Schedule remote agents (feature-gated) |
| `claude-api` | claude-api | — | Claude API access skill (feature-gated) |
| `claude-in-chrome` | claude-in-chrome | — | Browser integration (feature-gated) |
| `run-skill-generator` | — | — | Skill generation utility (feature-gated) |

---

## Plugin Skills & MCP Skills

**Source**: `src/plugins/`, `src/services/mcp/`

### Plugin Skills

Plugins can register skills via two mechanisms:

1. **Bundled-style registration**: `BundledSkillDefinition` → `loadedFrom: 'bundled'`
2. **Directory skills**: Plugins include a `skills/` directory → `loadedFrom: 'plugin'`

### MCP Skills

MCP servers can expose skills via the tool protocol. Key differences from file-based skills:

- `isMcp: true` on the command
- `loadedFrom: 'mcp'`
- **Shell commands (`!`\`...\`) are NOT executed** in MCP skills (security restriction)
- Registered via `mcpSkillBuilders` (write-once registry)

```typescript
// MCP skills skip inline shell execution:
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(...)
}
```

---

## Command Registry & Aggregation

**Source**: `src/commands.ts`

The command registry is the central hub that merges all sources into a single ordered list.

### Aggregation Order

```typescript
export async function getCommands(cwd: string): Promise<Command[]> {
  const {
    skillDirCommands,
    pluginSkills,
    bundledSkills,
    builtinPluginSkills,
  } = await getSkills(cwd)

  return [
    ...bundledSkills,          // 1. Bundled (compiled into binary)
    ...builtinPluginSkills,    // 2. Built-in plugin skills
    ...skillDirCommands,       // 3. Disk-based SKILL.md files
    ...workflowCommands,       // 4. Workflow commands (feature-gated)
    ...pluginCommands,         // 5. Plugin commands (non-skill)
    ...pluginSkills,           // 6. Plugin skills
    ...dynamicSkills,          // 7. Conditionally activated skills
    ...COMMANDS(),             // 8. Built-in hardcoded slash commands
  ]
}
```

### Finding a Command

```typescript
export function findCommand(
  commandName: string,
  commands: Command[],
): Command | undefined {
  // Matches by name or aliases[]
}
```

---

## SkillTool — How the Model Invokes Skills

**Source**: `src/tools/SkillTool/SkillTool.ts`

The `SkillTool` is the bridge that allows the model to invoke skills programmatically — without the user typing `/skill-name`.

### Input / Output Schema

```typescript
// Input
const inputSchema = z.object({
  skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
  args: z.string().optional().describe('Optional arguments for the skill'),
})

// Output (inline skill)
type InlineOutput = {
  success: boolean
  commandName: string
  allowedTools?: string[]
  model?: string
  status?: 'inline'
}

// Output (forked skill)
type ForkedOutput = {
  success: boolean
  commandName: string
  status: 'forked'
  agentId: string
  result: string  // The sub-agent's final response text
}
```

### SkillTool System Prompt to Model (verbatim from source)

> `/<skill-name> (e.g., /commit) is shorthand for users to invoke a user-invocable skill.`
> `When executed, the skill gets expanded to a full prompt.`
> `Use the Skill tool to execute them.`
> `IMPORTANT: Only use Skill for skills listed in its user-invocable skills section`
> `- do not guess or use built-in CLI commands.`

### getAllCommands (SkillTool's view)

```typescript
async function getAllCommands(context: ToolUseContext): Promise<Command[]> {
  // Returns getSkillToolCommands() — filtered for model-invocable skills
}
```

---

## Slash Command Invocation (User-Facing)

**Source**: `src/utils/slashCommandParsing.ts`, `src/utils/processUserInput/processSlashCommand.tsx`

### Parsing

```typescript
export function parseSlashCommand(input: string): ParsedSlashCommand | null

export type ParsedSlashCommand = {
  commandName: string  // "commit" from "/commit -am 'message'"
  args: string         // "-am 'message'"
  isMcp: boolean       // true if from MCP server
}
```

### System Prompt Injection

At session start, the model receives the list of user-invocable slash commands:

```typescript
// src/utils/messages/systemInit.ts
slash_commands: inputs.commands
  .filter(c => c.userInvocable !== false)
  .map(c => c.name)
```

This is why the model knows which `/commands` are available to mention to users.

### Forked Slash Command Execution

```typescript
async function executeForkedSlashCommand(
  command: CommandBase & PromptCommand,
  args: string,
  context: ProcessUserInputContext,
  precedingInputBlocks: ContentBlockParam[],
  setToolJSX: SetToolJSXFn,
  canUseTool: CanUseToolFn,
): Promise<SlashCommandResult> {
  // 1. Prepare forked command context
  // 2. Merge skill's effort into agent definition
  // 3. Launch sub-agent via runAgent()
  // 4. Collect messages from sub-agent
  // 5. Extract result text and return
}
```

---

## Execution Modes: Inline vs Fork

**Source**: `src/tools/SkillTool/SkillTool.ts`, `src/types/command.ts`

### Inline (default, `context: inline`)

The skill's expanded prompt is injected directly into the **current conversation** as a user message. The model then responds in the same context, with access to all conversation history.

```
User: /commit
  ↓
Skill expands: "Create a git commit. Follow conventional commits format..."
  ↓
Model responds in same conversation thread
```

### Fork (`context: fork`)

The skill runs in an **isolated sub-agent** with a separate token budget. The parent conversation waits for the sub-agent to complete, then receives a summary.

```
User: /review-pr 123
  ↓
Skill spawns sub-agent with forked context
  ↓
Sub-agent runs with isolated history + token budget
  ↓
Sub-agent produces final response
  ↓
Result returned to parent conversation
```

```typescript
async function executeForkedSkill(
  command: Command & { type: 'prompt' },
  commandName: string,
  args: string | undefined,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<Progress>,
): Promise<ToolResult<Output>>
```

The `agent` frontmatter field specifies the agent type for forked execution (e.g., `agent: code-reviewer`).

---

## Prompt Expansion Pipeline

**Source**: `src/skills/loadSkillsDir.ts` — `getPromptForCommand()`

When a skill is invoked, its markdown content goes through this pipeline before being injected:

```
Raw SKILL.md content
    │
    ▼
1. Prepend baseDir (if skill has a directory)
   "Base directory for this skill: /path/to/skill/\n\n<content>"
    │
    ▼
2. Argument substitution
   $ARGUMENTS, $ARGUMENTS[0], $0, $argname → actual arg values
    │
    ▼
3. ${CLAUDE_SKILL_DIR} substitution
   → replaced with absolute path to skill's directory
   → Windows: backslashes converted to forward slashes
    │
    ▼
4. ${CLAUDE_SESSION_ID} substitution
   → replaced with current session UUID
    │
    ▼
5. Inline shell execution (non-MCP skills only)
   !`command` → executed, output inlined
    │
    ▼
Final ContentBlockParam[] injected into conversation
```

### Inline Shell Commands

Skills can embed shell commands that execute at expansion time:

```markdown
Current git status:
!`git status --short`

Recent commits:
!`git log --oneline -10`
```

The backtick syntax `!`\`command\`` executes the shell command and replaces it with the output. This does **not** run for MCP-sourced skills.

Shell mode is controlled by the `shell` frontmatter key:
- `inherit` — inherits user's shell
- `no-shell` — exec without shell
- `restrict` — restricted shell mode

---

## Argument Substitution System

**Source**: `src/utils/argumentSubstitution.ts`

### Placeholder Syntax

| Placeholder | Resolves to |
|-------------|------------|
| `$ARGUMENTS` | Full args string |
| `$ARGUMENTS[0]` | First argument |
| `$ARGUMENTS[1]` | Second argument |
| `$0` | First argument (shorthand) |
| `$1` | Second argument (shorthand) |
| `$argname` | Named argument (from `arguments: argname` frontmatter) |

### Named Arguments Example

Frontmatter:
```yaml
arguments: branch_name ticket_id
argument-hint: "[branch] [ticket]"
```

Content:
```markdown
Create branch: $branch_name
For ticket: $ticket_id
All remaining: $ARGUMENTS
```

Invocation: `/create-branch feature/auth JIRA-123`

Expands to:
```
Create branch: feature/auth
For ticket: JIRA-123
All remaining: feature/auth JIRA-123
```

### Key Functions

```typescript
export function parseArguments(args: string): string[]
// Uses shell-quote: handles quoted strings, escaped chars

export function parseArgumentNames(
  argumentNames: string | string[] | undefined,
): string[]
// Parses "arg1 arg2" or ["arg1", "arg2"]

export function generateProgressiveArgumentHint(
  argNames: string[],
  typedArgs: string[],
): string | undefined
// Returns "[arg2] [arg3]" showing remaining args not yet typed

export function substituteArguments(
  content: string,
  args: string | undefined,
  appendIfNoPlaceholder = true,  // Append args to end if no $ARGUMENTS placeholder
  argumentNames: string[] = [],
): string
```

---

## Conditional Skills (Path-Triggered)

**Source**: `src/skills/loadSkillsDir.ts`

Skills with a `paths:` frontmatter field are **conditional** — they only activate when the user edits files matching those patterns.

```yaml
---
description: TypeScript-specific review checklist
paths:
  - "**/*.ts"
  - "**/*.tsx"
---
```

### How It Works

1. At load time, skills with `paths:` are stored in a `conditionalSkills` map (not `activeSkills`)
2. When the user's tool calls touch files (Bash, Edit, Write, etc.), `activateConditionalSkillsForPaths()` is called
3. If any file path matches a conditional skill's patterns, that skill moves to `dynamicSkills`
4. `getDynamicSkills()` returns the currently activated set

```typescript
export async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string,
): Promise<string[]>

export async function addSkillDirectories(dirs: string[]): Promise<void>

export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[]  // Returns names of newly activated skills

export function getDynamicSkills(): Command[]
```

Pattern matching uses the **`ignore` library** (same syntax as `.gitignore`).

---

## Skill Visibility Filters

**Source**: `src/commands.ts`

There are two filtered views of the command registry, each serving a different consumer.

### `getSkillToolCommands` — For the Model (SkillTool)

```typescript
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> =>
    allCommands.filter(cmd =>
      cmd.type === 'prompt' &&
      !cmd.disableModelInvocation &&
      cmd.source !== 'builtin' &&
      (
        cmd.loadedFrom === 'bundled' ||
        cmd.loadedFrom === 'skills' ||
        cmd.loadedFrom === 'commands_DEPRECATED' ||
        cmd.hasUserSpecifiedDescription ||
        cmd.whenToUse
      )
    )
)
```

A skill appears in SkillTool if:
- It's a `PromptCommand`
- `disableModelInvocation` is NOT set
- It's not a hardcoded `builtin`
- It has a user-specified description OR `whenToUse` field OR is bundled/disk-based

### `getSlashCommandToolSkills` — For Slash Command UI

```typescript
export const getSlashCommandToolSkills = memoize(
  async (cwd: string): Promise<Command[]> =>
    allCommands.filter(cmd =>
      cmd.type === 'prompt' &&
      cmd.source !== 'builtin' &&
      (cmd.hasUserSpecifiedDescription || cmd.whenToUse) &&
      (
        cmd.loadedFrom === 'skills' ||
        cmd.loadedFrom === 'plugin' ||
        cmd.loadedFrom === 'bundled' ||
        cmd.disableModelInvocation
      )
    )
)
```

---

## Context Window Budget for Skills

**Source**: `src/tools/SkillTool/prompt.ts`

The SkillTool listing in the system prompt is **budget-constrained** to avoid consuming too much context window space.

```typescript
const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 1% of context window
const MAX_LISTING_DESC_CHARS = 250
```

### Budget Algorithm

1. Try to fit all skills with full descriptions
2. If over budget, truncate non-bundled skill descriptions to `MAX_LISTING_DESC_CHARS`
3. Bundled skills are **never truncated** (always shown in full)
4. If still over budget after truncation, drop skills from the end

```typescript
export function formatCommandsWithinBudget(
  commands: Command[],
  contextWindowTokens?: number,
): string
```

Telemetry: `tengu_skill_descriptions_truncated` event fired when truncation occurs.

---

## Hooks in Skills

**Source**: `src/skills/loadSkillsDir.ts` — `parseHooksFromFrontmatter()`

Skills can register hooks that fire during their execution:

```yaml
hooks:
  PostToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: echo "Bash tool was used"
  PreToolUse:
    - matcher: Edit
      hooks:
        - type: command
          command: ./validate-edit.sh
```

### Hook Parsing

```typescript
function parseHooksFromFrontmatter(
  frontmatter: FrontmatterData,
  skillName: string,
): HooksSettings | undefined
```

**Security note**: Hooks in skills are gated — they require the same permission model as user-configured hooks. A skill cannot register hooks that execute arbitrary code without the user's permission model allowing it.

---

## Permission & Isolation Model

**Source**: `src/utils/permissions/filesystem.ts`

### Tool Allowlist

When a skill specifies `allowed-tools`, only those tools can be used during skill execution. The allowlist supports patterns:

```yaml
allowed-tools:
  - Read                    # Exact tool name
  - Bash(git log:*)         # Bash, but only commands starting with "git log"
  - Bash(grep:*)            # Bash grep commands only
  - Edit                    # Full Edit tool access
```

### Filesystem Permissions

Skills get scoped filesystem permissions based on their source:

| Skill source | Filesystem scope |
|-------------|-----------------|
| Global user (`~/.claude/skills/name/`) | `~/.claude/skills/name/**` |
| Project (`.claude/skills/name/`) | `.claude/skills/name/**` |

This means a skill can only access its own directory's files — it cannot read other skills or `.claude/settings.json`.

### Permission Decision Flow

```typescript
async checkPermissions(
  { skill, args },
  context,
): Promise<PermissionDecision> {
  // 1. Check deny rules (exact name match or prefix:*)
  // 2. Check allow rules (exact name match or prefix:*)
  // 3. Auto-allow if only safe/non-sensitive properties
  // 4. Return 'ask' for user permission or 'allow'/'deny'
}
```

---

## Skill Caching Architecture

**Source**: `src/commands.ts`

Multiple memoization layers prevent re-scanning skill directories on every command:

```
Layer 1: mcpSkillBuilders       ← write-once MCP registry
Layer 2: loadAllCommands        ← memoized by cwd (full aggregation)
Layer 3: getSkillToolCommands   ← memoized filtered view for model
Layer 4: getSlashCommandToolSkills ← memoized filtered view for slash UI
Layer 5: Skill index cache      ← experimental skill search (EXPERIMENTAL_SKILL_SEARCH feature)
```

### Cache Invalidation

```typescript
export function clearCommandsCache(): void {
  clearCommandMemoizationCaches()  // Clears layers 2-4
  clearPluginCommandCache()
  clearPluginSkillsCache()
  clearSkillCaches()               // Clears skill directory scan cache
}

export function clearCommandMemoizationCaches(): void {
  loadAllCommands.cache?.clear?.()
  getSkillToolCommands.cache?.clear?.()
  getSlashCommandToolSkills.cache?.clear?.()
  clearSkillIndexCache?.()
}
```

Cache is invalidated when:
- User installs/removes a plugin
- MCP servers change
- `--add-dir` is modified at runtime
- Dynamic skills activate (conditional path triggers)

---

## Skillify — The Meta Skill

**Source**: `src/skills/bundled/skillify.ts`

`/skillify` is a bundled skill that **creates skills from session workflows**. It is itself a skill that generates SKILL.md files.

```typescript
export function registerSkillifySkill(): void {
  registerBundledSkill({
    name: 'skillify',
    description: "Capture this session's repeatable process into a skill",
    allowedTools: ['Read', 'Write', 'Edit', 'Glob', 'Grep', 'AskUserQuestion'],
    userInvocable: true,
    disableModelInvocation: true,  // User-only, model cannot self-invoke
    getPromptForCommand(args, context) {
      // Returns complex interview prompt with session context
      // Guides model to:
      // 1. Interview user about the repeatable process
      // 2. Define argument names and types
      // 3. Choose inline vs fork execution
      // 4. Generate SKILL.md with correct frontmatter
      // 5. Save to appropriate location (~/.claude/skills/ or project)
    },
  })
}
```

The skillify prompt includes the current session context so the model can propose a skill that captures exactly what was just done.

---

## Full Data Flow Diagram

```
User types: /review-pr 456
    │
    ▼
parseSlashCommand()
  → { commandName: 'review-pr', args: '456' }
    │
    ▼
findCommand('review-pr', commands)
  → PromptCommand { context: 'fork', agent: 'code-reviewer', ... }
    │
    ├── context === 'inline'?
    │     ↓
    │   getPromptForCommand('456', ctx)
    │     ↓
    │   Expansion pipeline:
    │     1. prepend baseDir
    │     2. $ARGUMENTS → '456'
    │     3. ${CLAUDE_SKILL_DIR} → /path/to/skill/
    │     4. ${CLAUDE_SESSION_ID} → uuid
    │     5. !`shell cmds` → execute & inline
    │     ↓
    │   Inject into current conversation as user message
    │     ↓
    │   Model responds inline
    │
    └── context === 'fork'?
          ↓
        executeForkedSlashCommand()
          ↓
        runAgent(agentType='code-reviewer', prompt=expanded)
          ↓
        Sub-agent runs with isolated history + token budget
          ↓
        Sub-agent final response
          ↓
        Result returned to parent conversation

──────────────────────────────────────────────────────

Model invokes via SkillTool:
  { skill: 'commit', args: '-m "fix bug"' }
    │
    ▼
SkillTool.call()
    │
    ▼
getSkillToolCommands(cwd)
  → filtered list of model-invocable skills
    │
    ▼
findCommand('commit', ...)
    │
    ├── context !== 'fork' → inline expansion (same as above)
    │
    └── context === 'fork' → executeForkedSkill() → sub-agent
```

---

## Security Research Notes

1. **Inline shell execution** (`!`\`...\`): Skills can execute arbitrary shell commands at expansion time. Any user with write access to `~/.claude/skills/` or `.claude/skills/` can create a skill with malicious shell commands that execute when the skill is invoked — no separate permission prompt is shown for the shell commands themselves.

2. **MCP skill injection**: MCP servers can register skills (commands). A malicious MCP server could register a skill with a misleading description to get the model to invoke it. The skill's content is controlled by the MCP server.

3. **`paths:` conditional activation**: A skill can declare broad path patterns like `paths: ["**"]` and activate on any file operation, injecting its prompt into every subsequent context.

4. **`allowedTools` bypass consideration**: The tool allowlist (`allowed-tools`) restricts what tools the skill can call. However, an inline skill's expanded prompt becomes a user message — the model responds to it with its full tool set unless the allowlist is enforced at the execution engine level.

5. **`${CLAUDE_SESSION_ID}` exposure**: The current session ID is injected into every skill prompt that uses this placeholder. If a skill is controlled by a third party (plugin, MCP), they receive the session ID.

6. **Ancestor directory scanning**: Skill loading walks from `cwd` up to the home directory looking for `.claude/skills/` directories. A malicious ancestor directory (e.g., in a shared hosting environment) could inject skills into all child project sessions.

7. **`immediate: true` commands**: The `CommandBase` has an `immediate` field — commands with this set execute without confirmation. Worth investigating which commands use this.

8. **Hooks in skills**: If a skill can register `PostToolUse` or `PreToolUse` hooks, and those hooks run shell commands, a malicious skill could set up persistent hooks that execute on every tool call for the duration of the session.
