# Tools List — Claude Code

> Extracted from Claude Code npm source map leak (March 31, 2026).  
> 43 built-in tools + dynamic MCP tools. Static analysis only.

---

## Summary

| Category | Tools | Count |
|----------|-------|-------|
| File Operations | Read, Write, Edit, Glob, Grep | 5 |
| Shell | Bash, PowerShell | 2 |
| Code Intelligence | LSP | 1 |
| Notebooks | NotebookEdit | 1 |
| Web | WebFetch, WebSearch | 2 |
| Agent & Skills | Agent, SkillTool, ToolSearch | 3 |
| Planning & Modes | EnterPlanMode, ExitPlanMode | 2 |
| Worktree | EnterWorktree, ExitWorktree | 2 |
| MCP | MCPTool, ListMcpResources, ReadMcpResource, McpAuth | 4 |
| Tasks (V2) | TaskCreate, TaskGet, TaskList, TaskUpdate, TaskStop, TaskOutput | 6 |
| Todo (V1) | TodoWrite | 1 |
| Team / Swarm | TeamCreate, TeamDelete, SendMessage | 3 |
| Scheduling | CronCreate, CronDelete, CronList | 3 |
| Remote | RemoteTrigger | 1 |
| Async | Sleep | 1 |
| Configuration | Config | 1 |
| User Communication | Brief (SendUserMessage), AskUserQuestion | 2 |
| Internal | SyntheticOutput | 1 |
| **Total** | | **45** |

---

## File Operations

### Read
- **Name:** `Read`
- **Description:** Read a file from the local filesystem. Supports text, images (PNG/JPG), PDFs (up to 20 pages per request), and Jupyter notebooks. Returns content with line numbers in `cat -n` format.
- **Input:**
  - `file_path` (string, required) — absolute path
  - `offset` (number, optional) — start line
  - `limit` (number, optional) — max lines to read
  - `pages` (string, optional) — PDF page range e.g. `"1-5"`
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### Write
- **Name:** `Write`
- **Description:** Write or overwrite a file on the local filesystem. Requires the file to have been Read first if it already exists.
- **Input:**
  - `file_path` (string, required) — absolute path
  - `content` (string, required) — full file contents
- **Read-only:** No
- **Concurrency-safe:** No

---

### Edit
- **Name:** `Edit`
- **Description:** Perform exact string replacements in a file. Requires the file to have been Read first. The `old_string` must be unique in the file, or `replace_all` must be set.
- **Input:**
  - `file_path` (string, required)
  - `old_string` (string, required) — text to replace
  - `new_string` (string, required) — replacement text
  - `replace_all` (boolean, optional, default false) — replace every occurrence
- **Read-only:** No
- **Concurrency-safe:** No

---

### Glob
- **Name:** `Glob`
- **Description:** Fast file pattern matching. Returns matching file paths sorted by modification time. Supports patterns like `**/*.ts`, `src/**/*.tsx`.
- **Input:**
  - `pattern` (string, required) — glob pattern
  - `path` (string, optional) — directory to search in (default: cwd)
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### Grep
- **Name:** `Grep`
- **Description:** Search file contents using ripgrep regex. Supports context lines, file type filters, output modes (content / files_with_matches / count), multiline matching.
- **Input:**
  - `pattern` (string, required) — regex pattern
  - `path` (string, optional) — file or directory
  - `glob` (string, optional) — file filter e.g. `"*.ts"`
  - `type` (string, optional) — file type e.g. `"js"`, `"py"`
  - `output_mode` (enum, optional) — `content` | `files_with_matches` | `count`
  - `-A` / `-B` / `-C` / `context` (number, optional) — lines of context
  - `-n` (boolean, optional) — show line numbers
  - `-i` (boolean, optional) — case insensitive
  - `multiline` (boolean, optional) — allow `.` to match newlines
  - `head_limit` (number, optional, default 250) — limit output lines
  - `offset` (number, optional) — skip first N results
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Shell

### Bash
- **Name:** `Bash`
- **Description:** Execute a shell command in bash. Output is captured. Long-running commands can run in background. Sandboxed on supported platforms. Has tree-sitter AST security validation on dangerous patterns.
- **Input:**
  - `command` (string, required) — shell command to run
  - `timeout` (number, optional) — milliseconds, max 600000
  - `description` (string, optional) — human-readable label shown in UI
  - `run_in_background` (boolean, optional) — don't wait for completion
- **Read-only:** No (depends on command; read-only commands validated via allowlist)
- **Concurrency-safe:** No
- **Max output:** 30,000 characters

---

### PowerShell
- **Name:** `PowerShell`
- **Description:** Execute PowerShell commands. Supports both Windows PowerShell 5.1 (Desktop) and PowerShell 7+ (Core). Has equivalent security validation to BashTool.
- **Input:**
  - `command` (string, required)
  - `timeout` (number, optional) — milliseconds
  - `run_in_background` (boolean, optional)
- **Read-only:** No
- **Concurrency-safe:** No

---

## Code Intelligence

### LSP
- **Name:** `LSP`
- **Description:** Language Server Protocol operations — go to definition, find references, hover info, document/workspace symbols, go to implementation, call hierarchy. Requires an active LSP server in the IDE.
- **Input:**
  - `operation` (enum, required):
    - `goToDefinition` — find where a symbol is defined
    - `findReferences` — list all usages of a symbol
    - `hover` — get hover documentation
    - `documentSymbol` — list symbols in a file
    - `workspaceSymbol` — search symbols across workspace
    - `goToImplementation` — find implementations of an interface
    - `prepareCallHierarchy` — get call hierarchy root
    - `incomingCalls` — callers of a function
    - `outgoingCalls` — functions called by a function
  - `filePath` (string, required for most operations)
  - `line` (number, required for position-based ops)
  - `character` (number, required for position-based ops)
  - `query` (string, required for `workspaceSymbol`)
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Notebooks

### NotebookEdit
- **Name:** `NotebookEdit`
- **Description:** Edit Jupyter notebook cells (.ipynb files). Can replace, insert, or delete cells.
- **Input:**
  - `notebook_path` (string, required) — absolute path to .ipynb
  - `cell_id` (string, optional) — target cell ID
  - `new_source` (string, required) — cell content
  - `cell_type` (enum, optional) — `code` | `markdown`
  - `edit_mode` (enum, optional) — `replace` | `insert` | `delete`
- **Read-only:** No
- **Concurrency-safe:** No

---

## Web

### WebFetch
- **Name:** `WebFetch`
- **Description:** Fetch content from a URL and extract relevant information using a prompt to guide extraction.
- **Input:**
  - `url` (string, required)
  - `prompt` (string, required) — what to extract from the page
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### WebSearch
- **Name:** `WebSearch`
- **Description:** Search the web for current information. Returns search result snippets.
- **Input:**
  - `query` (string, required)
  - `allowed_domains` (string[], optional) — restrict results to these domains
  - `blocked_domains` (string[], optional) — exclude these domains
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Agent & Skills

### Agent
- **Name:** `Agent`
- **Description:** Launch a specialized sub-agent (subprocess) that autonomously handles complex tasks. Each agent type has specific capabilities and tools. Can run in the background or as an isolated git worktree.
- **Input:**
  - `subagent_type` (string, required) — agent type name (e.g. `general-purpose`, `Explore`, `Plan`)
  - `prompt` (string, required) — task for the agent
  - `description` (string, optional) — short label
  - `run_in_background` (boolean, optional)
  - `isolation` (enum, optional) — `"worktree"` for isolated git worktree
- **Read-only:** No
- **Concurrency-safe:** No
- **Built-in agent types:** `general-purpose`, `Explore`, `Plan`, `statusline-setup`, `claude-code-guide`, `verification`

---

### Skill
- **Name:** `Skill` (invoked as `/skill-name`)
- **Description:** Execute a skill loaded from a SKILL.md file or MCP prompt. Skills expand into prompt templates. Can run inline (in current context) or forked (as a sub-agent).
- **Input:**
  - `skill` (string, required) — skill name
  - `args` (string, optional) — arguments passed as `$ARGUMENTS`
- **Read-only:** Depends on skill
- **Concurrency-safe:** No

---

### ToolSearch
- **Name:** `ToolSearch`
- **Description:** Fetch schema definitions for deferred tools by name or keyword. Use `"select:ToolName"` for exact lookup or keywords for fuzzy search.
- **Input:**
  - `query` (string, required) — e.g. `"select:Read,Edit"` or `"notebook jupyter"`
  - `max_results` (number, optional, default 5)
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Planning & Modes

### EnterPlanMode
- **Name:** `EnterPlanMode`
- **Description:** Switch to plan mode — a read-only exploration and design phase. Claude can only read files and write to the plan file. Triggers a 5-phase workflow (or interview phase). Disabled when `--channels` is active.
- **Input:** *(none)*
- **Read-only:** Yes (tool itself is read-only)
- **Concurrency-safe:** Yes

---

### ExitPlanMode
- **Name:** `ExitPlanMode`
- **Description:** Present the completed plan to the user and request approval to exit plan mode and begin implementation. Restores the previous permission mode. Validates that plan mode is currently active. Disabled when `--channels` is active.
- **Input:** *(empty object — plan content injected from disk before hooks/SDK see it)*
- **Read-only:** No (writes plan file)
- **Concurrency-safe:** Yes

---

## Worktree

### EnterWorktree
- **Name:** `EnterWorktree`
- **Description:** Create an isolated git worktree and switch the session into it. Symlinks `node_modules` and similar directories to avoid disk bloat.
- **Input:**
  - `name` (string, optional) — worktree slug (alphanumeric + `.` `-` `_`, max 64 chars)
- **Read-only:** No
- **Concurrency-safe:** No

---

### ExitWorktree
- **Name:** `ExitWorktree`
- **Description:** Exit the current worktree and return to the original directory.
- **Input:**
  - `action` (enum, required) — `keep` | `remove`
  - `discard_changes` (boolean, optional) — force remove even with uncommitted changes
- **Read-only:** No
- **Concurrency-safe:** Yes

---

## MCP (Model Context Protocol)

### MCPTool
- **Name:** `mcp__{server}__{tool}` (dynamic, one per MCP tool)
- **Description:** Passthrough proxy to an MCP server tool. Schema defined by the MCP server at runtime (JSON Schema, not Zod). Permissions are checked via `checkPermissions: passthrough`.
- **Input:** Defined by the MCP server's tool schema
- **Read-only:** Determined by MCP tool annotations (`readOnlyHint`)
- **Concurrency-safe:** No

---

### ListMcpResources
- **Name:** `ListMcpResourcesTool`
- **Description:** List available resources from connected MCP servers.
- **Input:**
  - `server` (string, optional) — filter by server name
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### ReadMcpResource
- **Name:** `ReadMcpResourceTool`
- **Description:** Read the content of a specific MCP resource by URI.
- **Input:**
  - `server` (string, required) — MCP server name
  - `uri` (string, required) — resource URI
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### McpAuth
- **Name:** `McpAuth` (pseudo-tool, not sent to API)
- **Description:** Internal tool used to complete OAuth authentication for MCP servers that return a 401. Triggers the auth flow and retries the original request. Never appears in Claude's tool list.
- **Input:** Internal only
- **Read-only:** No
- **Concurrency-safe:** No

---

## Tasks (V2 System)

### TaskCreate
- **Name:** `TaskCreate`
- **Description:** Create a new task in the V2 task management system.
- **Input:**
  - `subject` (string, required)
  - `description` (string, optional)
  - `activeForm` (object, optional) — structured form data
  - `metadata` (Record<string, unknown>, optional)
- **Read-only:** No
- **Concurrency-safe:** Yes

---

### TaskGet
- **Name:** `TaskGet`
- **Description:** Retrieve a task by its ID.
- **Input:**
  - `taskId` (string, required)
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### TaskList
- **Name:** `TaskList`
- **Description:** List all tasks in the current session.
- **Input:** *(none)*
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### TaskUpdate
- **Name:** `TaskUpdate`
- **Description:** Update a task's subject, description, status, owner, blocking relationships, or metadata.
- **Input:**
  - `taskId` (string, required)
  - `subject` (string, optional)
  - `description` (string, optional)
  - `activeForm` (object, optional)
  - `status` (string, optional)
  - `owner` (string, optional)
  - `addBlocks` (string[], optional)
  - `addBlockedBy` (string[], optional)
  - `metadata` (Record<string, unknown>, optional)
- **Read-only:** No
- **Concurrency-safe:** Yes

---

### TaskStop
- **Name:** `TaskStop`
- **Description:** Stop a running background task.
- **Input:**
  - `task_id` (string, optional) — task ID to stop
  - `shell_id` (string, optional, deprecated)
- **Read-only:** No
- **Concurrency-safe:** Yes

---

### TaskOutput
- **Name:** `TaskOutput`
- **Description:** Retrieve the output / logs of a background task.
- **Input:**
  - `task_id` (string, required)
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Todo (V1 System)

### TodoWrite
- **Name:** `TodoWrite`
- **Description:** Write the session's task checklist (V1 only — replaced by TaskCreate/TaskList in V2). Used to track progress on multi-step work and show granular implementation steps.
- **Input:**
  - `todos` (TodoList array, required) — full replacement list with `id`, `content`, `status` (`pending` | `in_progress` | `completed`), `priority` (`high` | `medium` | `low`)
- **Read-only:** No
- **Concurrency-safe:** Yes

---

## Team / Swarm Coordination

### TeamCreate
- **Name:** `TeamCreate`
- **Description:** Create a named team for coordinating multiple parallel agents (swarm). Sets up shared directories and mailboxes.
- **Input:**
  - `team_name` (string, required)
  - `description` (string, optional)
  - `agent_type` (string, optional)
- **Read-only:** No
- **Concurrency-safe:** Yes

---

### TeamDelete
- **Name:** `TeamDelete`
- **Description:** Disband a swarm team and clean up its directories and mailboxes.
- **Input:** *(none)*
- **Read-only:** No
- **Concurrency-safe:** Yes

---

### SendMessage
- **Name:** `SendMessage`
- **Description:** Send a message to a teammate, broadcast to all agents, or send to a specific socket/bridge session. Used for inter-agent communication in swarm setups.
- **Input:**
  - `to` (string, required) — recipient: teammate name, `"*"` for broadcast, `"uds:<socket-path>"`, or `"bridge:<session-id>"`
  - `message` (string or object, required)
  - `summary` (string, optional) — short label for UI
- **Read-only:** No
- **Concurrency-safe:** Yes

---

## Scheduling

### CronCreate
- **Name:** `CronCreate`
- **Description:** Schedule a recurring or one-shot prompt using a cron expression. Returns a job ID. Durable jobs persist across restarts (stored in `.claude/scheduled_tasks.json`); session-only jobs live in memory.
- **Input:**
  - `cron` (string, required) — 5-field cron expression e.g. `"0 9 * * 1"`
  - `prompt` (string, required) — prompt to run on schedule
  - `recurring` (boolean, optional, default true) — false = run once
  - `durable` (boolean, optional, default false) — persist across restarts
- **Read-only:** No
- **Concurrency-safe:** Yes

---

### CronDelete
- **Name:** `CronDelete`
- **Description:** Cancel a scheduled cron job by ID. Removes from `.claude/scheduled_tasks.json` (durable) or in-memory store (session-only).
- **Input:**
  - `job_id` (string, required)
- **Read-only:** No
- **Concurrency-safe:** Yes

---

### CronList
- **Name:** `CronList`
- **Description:** List all active cron jobs in the current session, including both durable and session-only jobs with their cron expressions and next run times.
- **Input:** *(none)*
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Remote

### RemoteTrigger
- **Name:** `RemoteTrigger`
- **Description:** Manage scheduled remote agent triggers via the Anthropic API (`/v1/code/triggers`). Supports list, get, create, update, and manual run.
- **Input:**
  - `action` (enum, required) — `list` | `get` | `create` | `update` | `run`
  - `trigger_id` (string, optional) — required for `get`, `update`, `run`
  - `body` (object, optional) — payload for `create` / `update`
- **Read-only:** No (`list`/`get` are read-only; `create`/`update`/`run` are not)
- **Concurrency-safe:** Yes

---

## Async

### Sleep
- **Name:** `Sleep`
- **Description:** Wait for a specified duration without holding a shell process. The user can interrupt at any time. Prefers this over `Bash(sleep ...)`. Can run concurrently with other tools.
- **Input:**
  - `duration` (number, required) — milliseconds to sleep
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Configuration

### Config
- **Name:** `Config`
- **Description:** Get or set Claude Code configuration settings — theme, model, permission rules, voice mode, and other session settings.
- **Input:**
  - `setting` (string, required) — setting key
  - `value` (any, optional) — if omitted, reads current value; if provided, sets it
- **Read-only:** No (write if value is provided)
- **Concurrency-safe:** Yes

---

## User Communication

### Brief (SendUserMessage)
- **Name:** `Brief`
- **Description:** Send a proactive message to the user without waiting for a turn. Used to communicate status, ask questions, or share file attachments mid-task.
- **Input:**
  - `message` (string, required) — markdown-formatted message
  - `attachments` (string[], optional) — file paths to attach
  - `status` (enum, optional) — `normal` | `proactive`
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

### AskUserQuestion
- **Name:** `AskUserQuestion`
- **Description:** Ask the user a question and block until they respond. Used in the plan mode interview phase and when clarification is needed before proceeding.
- **Input:**
  - `question` (string, required)
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Internal

### SyntheticOutput
- **Name:** `SyntheticOutput` (also called `StructuredOutput`)
- **Description:** Return the final response as structured JSON. Used in non-interactive / SDK sessions where a machine-readable output format is expected. Not available in interactive REPL sessions.
- **Input:** Any object (passthrough schema — caller defines the shape)
- **Read-only:** Yes
- **Concurrency-safe:** Yes

---

## Tool Properties Summary

| Tool | Read-only | Concurrency-safe | Available in Plan Mode |
|------|-----------|-----------------|----------------------|
| Read | ✅ | ✅ | ✅ |
| Write | ❌ | ❌ | ❌ (plan file only) |
| Edit | ❌ | ❌ | ❌ (plan file only) |
| Glob | ✅ | ✅ | ✅ |
| Grep | ✅ | ✅ | ✅ |
| Bash | ❌ | ❌ | ⚠️ read-only cmds only |
| PowerShell | ❌ | ❌ | ⚠️ read-only cmds only |
| LSP | ✅ | ✅ | ✅ |
| NotebookEdit | ❌ | ❌ | ❌ |
| WebFetch | ✅ | ✅ | ✅ |
| WebSearch | ✅ | ✅ | ✅ |
| Agent | ❌ | ❌ | ❌ (plan agents only) |
| Skill | — | ❌ | — |
| ToolSearch | ✅ | ✅ | ✅ |
| EnterPlanMode | ✅ | ✅ | N/A |
| ExitPlanMode | ❌ | ✅ | ✅ (only valid in plan) |
| EnterWorktree | ❌ | ❌ | ❌ |
| ExitWorktree | ❌ | ✅ | ❌ |
| MCPTool | — | ❌ | — |
| ListMcpResources | ✅ | ✅ | ✅ |
| ReadMcpResource | ✅ | ✅ | ✅ |
| McpAuth | ❌ | ❌ | — |
| TaskCreate | ❌ | ✅ | ❌ |
| TaskGet | ✅ | ✅ | ✅ |
| TaskList | ✅ | ✅ | ✅ |
| TaskUpdate | ❌ | ✅ | ❌ |
| TaskStop | ❌ | ✅ | ❌ |
| TaskOutput | ✅ | ✅ | ✅ |
| TodoWrite | ❌ | ✅ | ❌ |
| TeamCreate | ❌ | ✅ | ❌ |
| TeamDelete | ❌ | ✅ | ❌ |
| SendMessage | ❌ | ✅ | ❌ |
| CronCreate | ❌ | ✅ | ❌ |
| CronDelete | ❌ | ✅ | ❌ |
| CronList | ✅ | ✅ | ✅ |
| RemoteTrigger | — | ✅ | ❌ |
| Sleep | ✅ | ✅ | ✅ |
| Config | — | ✅ | — |
| Brief | ✅ | ✅ | ✅ |
| AskUserQuestion | ✅ | ✅ | ✅ |
| SyntheticOutput | ✅ | ✅ | — |
