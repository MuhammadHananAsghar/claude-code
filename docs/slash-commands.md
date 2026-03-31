# Slash Commands — Claude Code

> Extracted from Claude Code npm source map leak (March 31, 2026).  
> Static analysis only. Commands are prefixed with `/` when typed in the REPL.

---

## Command Types

| Type | Description |
|------|-------------|
| `local` | Runs JavaScript/React logic directly in the CLI process |
| `prompt` | Expands into a prompt string sent to the model |
| `skill` | SKILL.md-backed prompt template (user or bundled) |

---

## Availability

Some commands are restricted by auth context:

| Tag | Meaning |
|-----|---------|
| *(no tag)* | Available to all users |
| `[claude.ai]` | Requires claude.ai OAuth subscription |
| `[console]` | Requires direct Anthropic API key (Console users) |
| `[ant-only]` | Only available to Anthropic employees (`USER_TYPE=ant`) |
| `[feature-gated]` | Behind a GrowthBook feature flag or `bun:bundle` compile-time flag |

---

## All Commands (A–Z)

### add-dir
- **Description:** Add a new working directory to the current session context
- **Type:** local
- **Source:** `src/commands/add-dir/`

---

### advisor
- **Description:** Configure the advisor model
- **Type:** local
- **Source:** `src/commands/advisor.ts`

---

### agents
- **Description:** Manage agent configurations — list, create, edit, delete custom agents
- **Type:** local
- **Source:** `src/commands/agents/`

---

### ant-trace `[ant-only]`
- **Description:** Internal tracing / diagnostics tool for Anthropic employees
- **Type:** local
- **Source:** `src/commands/ant-trace/`

---

### autofix-pr `[ant-only]`
- **Description:** Automatically fix issues in a pull request
- **Type:** local
- **Source:** `src/commands/autofix-pr/`

---

### backfill-sessions `[ant-only]`
- **Description:** Backfill historical session data
- **Type:** local
- **Source:** `src/commands/backfill-sessions/`

---

### branch
- **Description:** Create a branch of the current conversation at this point — forks the session into a new timeline
- **Type:** local
- **Source:** `src/commands/branch/`

---

### bridge `[feature-gated: BRIDGE_MODE]`
- **Description:** Connect this terminal for remote-control sessions (CCR v1 environment registration)
- **Type:** local
- **Source:** `src/commands/bridge/`

---

### bridge-kick `[ant-only]`
- **Description:** Inject bridge failure states for manual recovery testing
- **Type:** local
- **Source:** `src/commands/bridge-kick.ts`

---

### brief `[feature-gated: KAIROS / KAIROS_BRIEF]`
- **Description:** Toggle brief-only mode (proactive messages only, no full responses)
- **Type:** local
- **Source:** `src/commands/brief.ts`

---

### btw `[ant-only]`
- **Description:** Internal Anthropic command
- **Type:** local
- **Source:** `src/commands/btw/`

---

### bughunter `[ant-only]`
- **Description:** Internal bug hunting workflow
- **Type:** local
- **Source:** `src/commands/bughunter/`

---

### chrome
- **Description:** Claude in Chrome (Beta) settings — set up and configure the Chrome extension integration
- **Type:** local
- **Source:** `src/commands/chrome/`

---

### clear
- **Description:** Clear conversation history and free up context. Resets the message history while keeping the session alive.
- **Type:** local
- **Source:** `src/commands/clear/`

---

### color
- **Description:** Set the prompt bar color for this session
- **Type:** local
- **Source:** `src/commands/color/`

---

### commit `[ant-only]`
- **Description:** Create a git commit
- **Type:** local
- **Source:** `src/commands/commit.ts`

---

### commit-push-pr `[ant-only]`
- **Description:** Commit, push, and open a pull request in one step
- **Type:** local
- **Source:** `src/commands/commit-push-pr.ts`

---

### compact
- **Description:** Clear conversation history and free up context, keeping a summary. Uses the model to summarize the conversation before clearing.
- **Type:** local
- **Source:** `src/commands/compact/`

---

### config
- **Description:** Open the config panel — view and modify Claude Code settings (theme, model, permissions, etc.)
- **Type:** local
- **Source:** `src/commands/config/`

---

### context
- **Description:** Show current context usage — token count, files in context, memory files loaded
- **Type:** local (also has `contextNonInteractive` variant)
- **Source:** `src/commands/context/`

---

### copy
- **Description:** Copy the last response to clipboard
- **Type:** local
- **Source:** `src/commands/copy/`

---

### cost
- **Description:** Show the total cost and duration of the current session
- **Type:** local
- **Source:** `src/commands/cost/`

---

### ctx_viz `[ant-only]`
- **Description:** Context visualization — visual representation of the context window state
- **Type:** local
- **Source:** `src/commands/ctx_viz/`

---

### debug-tool-call `[ant-only]`
- **Description:** Debug a specific tool call by ID
- **Type:** local
- **Source:** `src/commands/debug-tool-call/`

---

### desktop
- **Description:** Continue the current session in Claude Desktop — shows QR code or deep-link to hand off
- **Type:** local
- **Source:** `src/commands/desktop/`

---

### diff
- **Description:** Show a diff of changes made in the current session
- **Type:** local
- **Source:** `src/commands/diff/`

---

### doctor
- **Description:** Diagnose and verify your Claude Code installation and settings — checks auth, model, tools, MCP servers
- **Type:** local
- **Source:** `src/commands/doctor/`

---

### effort
- **Description:** Set effort level for model usage — controls extended thinking token budget
- **Type:** local
- **Source:** `src/commands/effort/`

---

### env `[ant-only]`
- **Description:** Manage environment variables for the session
- **Type:** local
- **Source:** `src/commands/env/`

---

### exit
- **Description:** Exit the REPL
- **Type:** local
- **Source:** `src/commands/exit/`

---

### export
- **Description:** Export the current conversation to a file or clipboard (Markdown format)
- **Type:** local
- **Source:** `src/commands/export/`

---

### extra-usage `[claude.ai]`
- **Description:** Configure extra usage to keep working when rate limits are hit (also has `extraUsageNonInteractive` variant)
- **Type:** local
- **Source:** `src/commands/extra-usage/`

---

### fast
- **Description:** Toggle fast mode — switches to a faster model variant for the session
- **Type:** local
- **Source:** `src/commands/fast/`

---

### feedback
- **Description:** Submit feedback about Claude Code
- **Type:** local
- **Source:** `src/commands/feedback/`

---

### files
- **Description:** List all files currently in context
- **Type:** local
- **Source:** `src/commands/files/`

---

### fork `[feature-gated: FORK_SUBAGENT]`
- **Description:** Fork the current conversation as a sub-agent
- **Type:** local
- **Source:** `src/commands/fork/`

---

### good-claude `[ant-only]`
- **Description:** Internal Anthropic positive reinforcement command
- **Type:** local
- **Source:** `src/commands/good-claude/`

---

### heapdump `[ant-only]`
- **Description:** Dump the JS heap to ~/Desktop for memory profiling
- **Type:** local
- **Source:** `src/commands/heapdump/`

---

### help
- **Description:** Show help and available commands — lists all slash commands with descriptions
- **Type:** local
- **Source:** `src/commands/help/`

---

### hooks
- **Description:** Manage allow & deny tool permission rules and configure hooks
- **Type:** local
- **Source:** `src/commands/hooks/`

---

### ide
- **Description:** Manage IDE integrations and show status — configure VS Code, JetBrains extensions
- **Type:** local
- **Source:** `src/commands/ide/`

---

### init
- **Description:** Initialize Claude Code in the current project — creates CLAUDE.md
- **Type:** local
- **Source:** `src/commands/init.ts`

---

### init-verifiers `[ant-only]`
- **Description:** Initialize verifier agents for a project
- **Type:** local
- **Source:** `src/commands/init-verifiers.js`

---

### insights
- **Description:** Generate a report analyzing your Claude Code sessions — usage patterns, activity breakdown
- **Type:** prompt (lazy-loaded from `src/commands/insights.ts`)
- **Source:** `src/commands.ts` (inline lazy shim)

---

### install
- **Description:** Install Claude Code native build
- **Type:** local
- **Source:** `src/commands/install.tsx`

---

### install-github-app
- **Description:** Set up Claude GitHub Actions for a repository
- **Type:** local
- **Source:** `src/commands/install-github-app/`

---

### install-slack-app
- **Description:** Install the Claude Slack app for your organization
- **Type:** local
- **Source:** `src/commands/install-slack-app/`

---

### issue `[ant-only]`
- **Description:** Open a GitHub issue for Claude CLI internal repo
- **Type:** local
- **Source:** `src/commands/issue/`

---

### keybindings
- **Description:** Open or create your keybindings configuration file
- **Type:** local
- **Source:** `src/commands/keybindings/`

---

### login `[claude.ai or console]`
- **Description:** Sign in to your Anthropic account (OAuth or API key)
- **Type:** local
- **Source:** `src/commands/login/`

---

### logout `[claude.ai or console]`
- **Description:** Sign out from your Anthropic account
- **Type:** local
- **Source:** `src/commands/logout/`

---

### mcp
- **Description:** Manage MCP servers — add, remove, list, configure Model Context Protocol servers
- **Type:** local
- **Source:** `src/commands/mcp/`

---

### memory
- **Description:** Edit Claude memory files — open CLAUDE.md or `~/.claude/memory/` files for editing
- **Type:** local
- **Source:** `src/commands/memory/`

---

### mobile
- **Description:** Show QR code to download the Claude mobile app
- **Type:** local
- **Source:** `src/commands/mobile/`

---

### mock-limits `[ant-only]`
- **Description:** Simulate rate limit states for testing
- **Type:** local
- **Source:** `src/commands/mock-limits/`

---

### model
- **Description:** Switch the active model for the session
- **Type:** local
- **Source:** `src/commands/model/`

---

### oauth-refresh `[ant-only]`
- **Description:** Force an OAuth token refresh
- **Type:** local
- **Source:** `src/commands/oauth-refresh/`

---

### onboarding `[ant-only]`
- **Description:** Trigger the new user onboarding flow
- **Type:** local
- **Source:** `src/commands/onboarding/`

---

### output-style
- **Description:** *(Deprecated)* Use `/config` to change output style instead
- **Type:** local
- **Source:** `src/commands/output-style/`

---

### passes `[claude.ai]`
- **Description:** Show plan usage limits and guest pass management
- **Type:** local
- **Source:** `src/commands/passes/`

---

### perf-issue `[ant-only]`
- **Description:** Report a performance issue with diagnostic data
- **Type:** local
- **Source:** `src/commands/perf-issue/`

---

### permissions
- **Description:** Manage allow & deny tool permission rules for the current project
- **Type:** local
- **Source:** `src/commands/permissions/`

---

### plan
- **Description:** Enable plan mode or view the current session plan. `/plan open` opens the plan file in `$EDITOR`.
- **Type:** local
- **Source:** `src/commands/plan/`

---

### plugin
- **Description:** Manage Claude Code plugins — install, remove, list plugins
- **Type:** local
- **Source:** `src/commands/plugin/`

---

### pr-comments
- **Description:** Get comments from a GitHub pull request and load them into context
- **Type:** local
- **Source:** `src/commands/pr_comments/`

---

### privacy-settings
- **Description:** Manage privacy settings — Grove (web search), data usage preferences
- **Type:** local
- **Source:** `src/commands/privacy-settings/`

---

### rate-limit-options
- **Description:** Show options when rate limit is reached — view upgrade paths, extra usage
- **Type:** local
- **Source:** `src/commands/rate-limit-options/`

---

### release-notes
- **Description:** Show release notes for the current version
- **Type:** local
- **Source:** `src/commands/release-notes/`

---

### reload-plugins
- **Description:** Activate pending plugin changes in the current session (hot reload)
- **Type:** local
- **Source:** `src/commands/reload-plugins/`

---

### remote-control `[claude.ai]` `[feature-gated: BRIDGE_MODE]`
- **Description:** Show remote session URL and QR code for connecting to this session from claude.ai
- **Alias:** `remote-control`
- **Type:** local
- **Source:** `src/commands/bridge/` (serves as the `/remote-control` entrypoint)

---

### remote-env `[claude.ai]`
- **Description:** Configure the default remote environment for teleport sessions
- **Type:** local
- **Source:** `src/commands/remote-env/`

---

### remote-setup `[feature-gated: CCR_REMOTE_SETUP]`
- **Description:** Web setup for remote Code Session (CCR v2)
- **Alias:** `web-setup`
- **Type:** local
- **Source:** `src/commands/remote-setup/`

---

### rename
- **Description:** Rename the current conversation
- **Type:** local
- **Source:** `src/commands/rename/`

---

### reset-limits `[ant-only]`
- **Description:** Reset rate limits for testing (also has `resetLimitsNonInteractive` variant)
- **Type:** local
- **Source:** `src/commands/reset-limits/`

---

### resume
- **Description:** Resume a previous conversation — pick from session history list
- **Type:** local
- **Source:** `src/commands/resume/`

---

### review `[claude.ai or console]`
- **Description:** Review a pull request — loads PR diff and comments into context for review
- **Type:** local
- **Source:** `src/commands/review.js`

---

### rewind
- **Description:** Rewind the conversation to a previous point — step back through message history
- **Type:** local
- **Source:** `src/commands/rewind/`

---

### sandbox
- **Description:** Toggle sandbox mode on/off for Bash command execution
- **Alias:** `sandbox-toggle`
- **Type:** local
- **Source:** `src/commands/sandbox-toggle/`

---

### security-review
- **Description:** Run a security review on the current codebase or specified files
- **Type:** local / prompt
- **Source:** `src/commands/security-review.js`

---

### session
- **Description:** Show remote session URL and QR code; manage the current session
- **Type:** local
- **Source:** `src/commands/session/`

---

### share `[ant-only]`
- **Description:** Share the current session (internal)
- **Type:** local
- **Source:** `src/commands/share/`

---

### skills
- **Description:** List available skills — shows all SKILL.md-backed slash commands
- **Type:** local
- **Source:** `src/commands/skills/`

---

### stats
- **Description:** Show your Claude Code usage statistics and activity summary
- **Type:** local
- **Source:** `src/commands/stats/`

---

### status
- **Description:** Show current Claude Code status — auth, model, context, tools
- **Type:** local
- **Source:** `src/commands/status/`

---

### statusline
- **Description:** Set up Claude Code's status line UI for shell prompts (bash/zsh/fish)
- **Type:** local
- **Source:** `src/commands/statusline.tsx`

---

### stickers
- **Description:** Order Claude Code stickers 🎉
- **Type:** local
- **Source:** `src/commands/stickers/`

---

### summary `[ant-only]`
- **Description:** Generate a summary of the current session
- **Type:** local
- **Source:** `src/commands/summary/`

---

### tag
- **Description:** Toggle a searchable tag on the current session for later filtering
- **Type:** local
- **Source:** `src/commands/tag/`

---

### tasks
- **Description:** List and manage background tasks — view running/completed tasks, stop tasks
- **Type:** local
- **Source:** `src/commands/tasks/`

---

### teleport `[ant-only]` `[claude.ai]`
- **Description:** Teleport (sync) the current session to a remote environment
- **Type:** local
- **Source:** `src/commands/teleport/`

---

### terminal-setup
- **Description:** Configure terminal integration — shell hooks, prompt customization
- **Alias:** `terminalSetup`
- **Type:** local
- **Source:** `src/commands/terminalSetup/`

---

### theme
- **Description:** Change the theme — light, dark, system, or custom color schemes
- **Type:** local
- **Source:** `src/commands/theme/`

---

### thinkback
- **Description:** Show the thinking animation / extended reasoning visualization
- **Alias:** `think-back`
- **Type:** local
- **Source:** `src/commands/thinkback/`

---

### thinkback-play
- **Description:** Play the thinkback animation
- **Type:** local
- **Source:** `src/commands/thinkback-play/`

---

### torch `[feature-gated: TORCH]`
- **Description:** Internal torch workflow command
- **Type:** local
- **Source:** `src/commands/torch.js`

---

### ultraplan `[ant-only]` `[feature-gated: ULTRAPLAN]`
- **Description:** Launch the ultra plan mode — extended multi-phase planning with subagents
- **Type:** local
- **Source:** `src/commands/ultraplan.js`

---

### ultrareview `[claude.ai]`
- **Description:** Deep AI code review using the Ultrareview quota — more thorough than `/review`
- **Type:** local
- **Source:** `src/commands/review.js` (exported as `ultrareview`)

---

### upgrade
- **Description:** Upgrade Claude Code to the latest version
- **Type:** local
- **Source:** `src/commands/upgrade/`

---

### usage
- **Description:** Show usage statistics — tokens used, cost breakdown, session history
- **Type:** local
- **Source:** `src/commands/usage/`

---

### version `[ant-only]`
- **Description:** Show detailed version information
- **Type:** local
- **Source:** `src/commands/version.js`

---

### vim
- **Description:** Toggle between Vim and Normal editing modes in the REPL input
- **Type:** local
- **Source:** `src/commands/vim/`

---

### voice `[feature-gated: VOICE_MODE]`
- **Description:** Toggle voice mode — speak to Claude and hear responses
- **Type:** local
- **Source:** `src/commands/voice/`

---

### workflows `[feature-gated: WORKFLOW_SCRIPTS]`
- **Description:** Run workflow scripts defined in `.claude/workflows/`
- **Type:** local
- **Source:** `src/commands/workflows/`

---

## Feature-Gated Commands (Compile-time flags)

These commands only exist in builds where the corresponding `bun:bundle` `feature()` flag is enabled:

| Command | Flag | Description |
|---------|------|-------------|
| `bridge` | `BRIDGE_MODE` | Remote control bridge daemon |
| `remote-control-server` | `DAEMON + BRIDGE_MODE` | Remote control server entrypoint |
| `voice` | `VOICE_MODE` | Voice input/output mode |
| `brief` | `KAIROS / KAIROS_BRIEF` | Brief-only mode toggle |
| `assistant` | `KAIROS` | Assistant command (KAIROS build) |
| `proactive` | `PROACTIVE / KAIROS` | Proactive suggestions mode |
| `fork` | `FORK_SUBAGENT` | Fork conversation as sub-agent |
| `workflows` | `WORKFLOW_SCRIPTS` | Run .claude/workflows/ scripts |
| `remote-setup` | `CCR_REMOTE_SETUP` | Web-based remote session setup |
| `ultraplan` | `ULTRAPLAN` | Extended ultra plan mode |
| `subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | Subscribe to PR webhook events |
| `torch` | `TORCH` | Internal torch workflow |
| `peers` | `UDS_INBOX` | Peer-to-peer Unix socket inbox |
| `buddy` | `BUDDY` | Buddy agent mode |
| `force-snip` | `HISTORY_SNIP` | Force history snipping |

---

## Dynamic Commands (Loaded at Runtime)

In addition to built-in commands, the following are loaded dynamically:

| Source | How loaded | When available |
|--------|-----------|----------------|
| **Bundled skills** | `getBundledSkills()` | Always — 17 built-in skills |
| **User SKILL.md files** | `getSkillDirCommands(cwd)` | When SKILL.md files found in project or `~/.claude/skills/` |
| **Plugin skills** | `getPluginSkills()` | When plugins with SKILL.md are installed |
| **Built-in plugin skills** | `getBuiltinPluginSkillCommands()` | When built-in plugins are enabled |
| **Plugin commands** | `getPluginCommands()` | When plugins with commands are installed |
| **Workflow scripts** | `getWorkflowCommands(cwd)` | When `.claude/workflows/*.md` files exist |
| **Dynamic skills** | `getDynamicSkills()` | Discovered during file operations |

---

## Command Count Summary

| Category | Count |
|----------|-------|
| Public (all users) | ~55 |
| claude.ai only | ~8 |
| console only | ~2 |
| ant-only (internal) | ~18 |
| Feature-gated | ~15 |
| Bundled skills | 17 |
| **Total built-in** | **~103+** |

Dynamic skills, plugin commands, and workflow scripts are additive on top of these.
