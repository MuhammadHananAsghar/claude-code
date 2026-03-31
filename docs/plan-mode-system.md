# Plan Mode System — Claude Code Internal Architecture

> Extracted from Claude Code npm source map leak (March 31, 2026).  
> Security research / bug bounty reference. Static analysis only — code cannot be run as-is.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Permission Mode Type System](#2-permission-mode-type-system)
3. [Entering Plan Mode](#3-entering-plan-mode)
4. [Plan Mode V2 — 5-Phase Workflow](#4-plan-mode-v2--5-phase-workflow)
5. [Interview Phase (Alternative Workflow)](#5-interview-phase-alternative-workflow)
6. [Dangerous Permission Stripping](#6-dangerous-permission-stripping)
7. [Tool Availability Matrix](#7-tool-availability-matrix)
8. [Plan File System](#8-plan-file-system)
9. [Plan Agents](#9-plan-agents)
10. [Exiting Plan Mode](#10-exiting-plan-mode)
11. [Attachment System](#11-attachment-system)
12. [Plan Mode State Machine](#12-plan-mode-state-machine)
13. [Team Coordination & Plan Approval](#13-team-coordination--plan-approval)
14. [Feature Flags & Experiments](#14-feature-flags--experiments)
15. [Agent Count by Subscription Tier](#15-agent-count-by-subscription-tier)
16. [CLI Activation](#16-cli-activation)
17. [UI Components](#17-ui-components)
18. [Session Persistence & Recovery](#18-session-persistence--recovery)
19. [handlePlanModeTransition](#19-handleplanmodetransition)
20. [Key Constants Reference](#20-key-constants-reference)
21. [File Inventory](#21-file-inventory)
22. [Security Findings](#22-security-findings)

---

## 1. Overview

Plan Mode is a **staged execution model** in Claude Code that separates the codebase exploration and design phase from the implementation phase. When active:

- Claude is restricted to **read-only operations** (no file writes, no code execution)
- Claude spawns dedicated **Explore** and **Plan** sub-agents to analyze the codebase
- Claude produces a **Markdown plan file** saved to `~/.claude/plans/`
- Claude must invoke `ExitPlanMode` — which requires **explicit user approval** — before any code changes

Plan Mode is gated by the `tengu_ccr_bridge` **GrowthBook** feature flag and has a V2 implementation (`planModeV2.ts`) that adds multi-agent parallelism and the interview phase.

**Core source files:**
- `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
- `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
- `src/utils/planModeV2.ts`
- `src/utils/plans.ts`
- `src/utils/attachments.ts`

---

## 2. Permission Mode Type System

**File:** `src/types/permissions.ts`

```typescript
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',
  'bypassPermissions',
  'default',
  'dontAsk',
  'plan',
] as const

export type ExternalPermissionMode = (typeof EXTERNAL_PERMISSION_MODES)[number]
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode
```

Plan is one of five **external** permission modes. Internally two additional modes exist (`auto` and `bubble`) used by the transcript classifier and agent delegation systems.

### ToolPermissionContext Fields Relevant to Plan

```typescript
interface ToolPermissionContext {
  mode: PermissionMode                            // current active mode
  prePlanMode?: PermissionMode                    // mode before entering plan (restored on exit)
  strippedDangerousRules?: ToolPermissionRulesBySource  // permissions disabled while in plan
}
```

---

## 3. Entering Plan Mode

**File:** `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`

### Tool Definition

```typescript
export const EnterPlanModeTool: Tool = buildTool({
  name: 'EnterPlanMode',
  searchHint: 'switch to plan mode to design an approach before coding',
  shouldDefer: true,
  isConcurrencySafe: true,
  isReadOnly: true,

  isEnabled() {
    // Disabled when --channels is active (no terminal for approval dialog)
    if (getAllowedChannels().length > 0) return false
    return true
  },

  async call(_input, context) {
    const appState = context.getAppState()
    handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')

    context.setAppState(prev => ({
      ...prev,
      toolPermissionContext: applyPermissionUpdate(
        prepareContextForPlanMode(prev.toolPermissionContext),
        { type: 'setMode', mode: 'plan', destination: 'session' },
      ),
    }))

    return { data: { message: 'Entered plan mode. Focus on exploring the codebase and designing an implementation approach.' } }
  },

  mapToolResultToToolResultBlockParam({ message }, toolUseID) {
    const instructions = isPlanModeInterviewPhaseEnabled()
      ? `${message}\n\nDO NOT write or edit any files except the plan file.`
      : `${message}\n\nRemember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.`

    return { type: 'tool_result', content: instructions, tool_use_id: toolUseID }
  },
})
```

### When Claude Should Enter Plan Mode

Per `src/tools/EnterPlanModeTool/prompt.ts`:

**For external users — enter when:**
1. New feature with unclear implementation approach
2. Multiple valid architectural choices exist
3. Code modifications that affect existing behavior
4. Major refactoring touching 2+ files
5. Requirements need exploration before a plan can be formed
6. User preferences are relevant to the approach taken

**Do NOT enter for:**
- Simple typo / one-liner fixes
- Trivial, obviously scoped implementations

---

## 4. Plan Mode V2 — 5-Phase Workflow

**File:** `src/utils/messages.ts` (function `getPlanModeV2Instructions`)

When plan mode is entered and the interview phase is **disabled**, Claude follows a structured 5-phase workflow:

```
Phase 1: Initial Understanding
  → Launch N Explore agents in parallel
  → Goal: comprehensive codebase comprehension

Phase 2: Design
  → Launch N Plan agents in parallel (each assigned a different perspective)
  → Goal: produce competing implementation designs in the plan file

Phase 3: Review
  → Claude (main loop) reviews all plans
  → Checks alignment, reconciles trade-offs

Phase 4: Final Plan
  → Format the final plan in the plan file
  → Content controlled by tengu_pewter_ledger experiment (size guidance)

Phase 5: Call ExitPlanMode
  → Present plan to user and await approval
  → On approval: restore previous permission mode + begin implementation
```

### Agent Types Used

| Phase | Agent Type | Count |
|-------|-----------|-------|
| Phase 1 | EXPLORE_AGENT | `getPlanModeV2ExploreAgentCount()` (default: 3) |
| Phase 2 | PLAN_AGENT | `getPlanModeV2AgentCount()` (1–3 by tier) |

---

## 5. Interview Phase (Alternative Workflow)

**File:** `src/utils/planModeV2.ts`

### Activation

```typescript
export function isPlanModeInterviewPhaseEnabled(): boolean {
  // Always enabled for Anthropic employees (ant)
  if (isAntUser()) return true
  // GrowthBook gate for external users
  if (feature('tengu_plan_mode_interview_phase')) return true
  // Env var override
  if (isEnvTruthy(process.env.CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE)) return true
  return false
}
```

### Interview Workflow

Instead of the sequential 5-phase model, the interview phase is **iterative and collaborative**:

```
Loop:
  1. Claude explores relevant parts of the codebase (using Explore agents or directly)
  2. Claude updates the plan file incrementally
  3. Claude calls AskUserQuestion to validate understanding / ask clarifying questions
  4. User responds
  5. Repeat until plan is complete and user approves
  → Claude calls ExitPlanMode
```

Instructions sent via `plan_mode` attachment:

```
- MUST NOT make any edits except to the plan file
- Ask targeted questions using AskUserQuestion
- Update plan file after each exploration round
- Do NOT dump all questions at once — iterative conversation
```

---

## 6. Dangerous Permission Stripping

**File:** `src/utils/permissions/permissionSetup.ts`

`prepareContextForPlanMode()` is called before the mode is set to `'plan'`. It handles the interaction between plan mode and auto mode (transcript classifier):

```typescript
export function prepareContextForPlanMode(
  context: ToolPermissionContext,
): ToolPermissionContext {
  if (context.mode === 'plan') return context  // already in plan mode

  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const planAutoMode = shouldPlanUseAutoMode()

    if (context.mode === 'auto') {
      if (planAutoMode) {
        // Keep auto mode active during plan, just save the pre-plan mode
        return { ...context, prePlanMode: 'auto' }
      }
      // Disable auto mode, restore dangerous permissions to safe state
      autoModeStateModule?.setAutoModeActive(false)
      setNeedsAutoModeExitAttachment(true)
      return {
        ...restoreDangerousPermissions(context),
        prePlanMode: 'auto',
      }
    }

    if (planAutoMode && context.mode !== 'bypassPermissions') {
      // Activate auto mode WITH dangerous permissions stripped
      autoModeStateModule?.setAutoModeActive(true)
      return {
        ...stripDangerousPermissionsForAutoMode(context),
        prePlanMode: context.mode,
      }
    }
  }

  return { ...context, prePlanMode: context.mode }
}
```

### What Counts as "Dangerous"

**Dangerous Bash rules** (blocked during auto mode in plan):

| Pattern | Reason |
|---------|--------|
| `Bash` with no content | Allows ALL bash commands |
| `Bash(*)` | Wildcard — all commands |
| `python:*`, `node:*` prefix syntax | Arbitrary script execution |
| `python*`, `python *`, `python -*` | Scripting patterns |

**Dangerous PowerShell rules:**

| Pattern | Reason |
|---------|--------|
| `PowerShell`, `PowerShell(*)` | All PowerShell |
| `Invoke-Expression`, `Invoke-Command` | Code injection |
| `Start-Process` | Process launch |
| `pwsh`, `cmd`, `wsl` | Nested shell access |

**Dangerous Agent rules:**
- `Agent(*)` — allows spawning arbitrary sub-agents

Stripped rules are saved in `strippedDangerousRules` on the context and restored when exiting plan mode.

---

## 7. Tool Availability Matrix

| Tool | Normal Mode | Plan Mode | Enforcement Mechanism |
|------|------------|----------|-----------------------|
| `EnterPlanMode` | Available | N/A (already in plan) | `validateInput` rejects |
| `ExitPlanMode` | Unavailable | Available | `isEnabled()` returns false outside plan |
| `Read` | Available | Available | Always read-only |
| `Glob` | Available | Available | Always read-only |
| `Grep` | Available | Available | Always read-only |
| `Bash` (read-only cmds) | Available | Available | Bash security validation |
| `Bash` (write/exec cmds) | Available | **BLOCKED** | Permission check + system prompt |
| `FileWrite` | Available | **BLOCKED** | System prompt + permission |
| `FileEdit` | Available | **BLOCKED** | System prompt + permission |
| `NotebookEdit` | Available | **BLOCKED** | Disallowed in plan agents |
| `Agent` | Available | **BLOCKED** in plan agents | `disallowedTools` list |
| `AskUserQuestion` | Available | Available | Used for interview phase |
| `WebSearch` / `WebFetch` | Available | Available | Read-only |

> **Note:** Tool filtering is not done at the API call layer. The model is guided by:
> 1. System prompt instructions ("MUST NOT modify files")
> 2. Plan agent `disallowedTools` list
> 3. Permission checks for file operations
> 4. Bash tool's own read-only validation

---

## 8. Plan File System

**File:** `src/utils/plans.ts`

### File Path Construction

```typescript
export function getPlanFilePath(agentId?: AgentId): string {
  const planSlug = getPlanSlug(getSessionId())

  if (!agentId) {
    // Main session: ~/.claude/plans/{slug}.md
    return join(getPlansDirectory(), `${planSlug}.md`)
  }
  // Sub-agent: ~/.claude/plans/{slug}-agent-{agentId}.md
  return join(getPlansDirectory(), `${planSlug}-agent-${agentId}.md`)
}

export function getPlanSlug(sessionId?: SessionId): string {
  // Lazy-generated, cached per session
  // Example output: "quantum-fox-beetle"
  const cache = getPlanSlugCache()
  let slug = cache.get(id)
  if (!slug) {
    slug = generateWordSlug()
    cache.set(id, slug)
  }
  return slug
}
```

### Plans Directory

```typescript
export function getPlansDirectory(): string {
  const settings = getInitialSettings()
  const settingsDir = settings.plansDirectory

  // User-specified (relative to project root) or default
  let plansPath = settingsDir
    ? resolve(getCwd(), settingsDir)
    : join(getClaudeConfigHomeDir(), 'plans')  // ~/.claude/plans

  // Path traversal defense: reject any path escaping cwd
  const resolved = resolve(plansPath)
  if (!resolved.startsWith(cwd + sep) && resolved !== cwd) {
    plansPath = join(getClaudeConfigHomeDir(), 'plans')
  }

  getFsImplementation().mkdirSync(plansPath)
  return plansPath
}
```

### Plan File Format

Plan files are **Markdown documents**. Plan agents are instructed to produce:

```markdown
## Critical Files for Implementation
- path/to/file1.ts
- path/to/file2.ts
- path/to/file3.ts

[Additional sections as needed:]
- Requirements and constraints
- Architecture decisions
- Step-by-step implementation strategy
- Dependencies and sequencing
- Potential challenges and mitigations
```

The `tengu_pewter_ledger` experiment controls Phase 4 guidance on plan length:
- **Control (`null`):** No size guidance
- **`trim`:** Light guidance to reduce verbosity
- **`cut`:** Moderate size constraints
- **`cap`:** Strict cap guidance

Baseline measurements: p50 = 4,906 chars, p90 = 11,617 chars. Goal: reduce size to lower API rejection rate (rejection rate climbs from ~20% at <2K chars to ~50% at >20K chars).

---

## 9. Plan Agents

**File:** `src/tools/AgentTool/built-in/planAgent.ts`

### PLAN_AGENT Definition

```typescript
export const PLAN_AGENT: BuiltInAgentDefinition = {
  agentType: 'Plan',
  whenToUse: 'Software architect agent for designing implementation plans',

  disallowedTools: [
    AGENT_TOOL_NAME,          // Cannot spawn nested agents
    EXIT_PLAN_MODE_TOOL_NAME, // Cannot exit plan mode directly
    FILE_EDIT_TOOL_NAME,      // No file editing (except plan file)
    FILE_WRITE_TOOL_NAME,     // No file creation (except plan file)
    NOTEBOOK_EDIT_TOOL_NAME,  // No notebook editing
  ],

  tools: EXPLORE_AGENT.tools, // Inherits all explore agent tools

  getSystemPrompt: () => getPlanV2SystemPrompt(),
}
```

### Plan Agent System Prompt

```
You are a software architect and planning specialist for Claude Code.
Your role is to explore the codebase and design implementation plans.

=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
STRICTLY PROHIBITED:
- Creating new files (no Write, touch, or file creation of any kind)
- Modifying existing files (no Edit operations)
- Deleting files (no rm or deletion)
- Moving or copying files (no mv or cp)
- Creating temporary files anywhere, including /tmp
- Using redirect operators (>, >>, |) or heredocs to write to files
- Running ANY commands that change system state

## Your Process
1. Understand Requirements — apply your assigned perspective
2. Explore Thoroughly — read files, find patterns, understand architecture
3. Design Solution — create implementation approach
4. Detail the Plan — step-by-step strategy, identify dependencies

## Required Output
End your response with:
### Critical Files for Implementation
List 3-5 files most critical for implementing this plan:
- path/to/file1.ts
- path/to/file2.ts

REMEMBER: You can ONLY explore and plan. You CANNOT modify any files.
```

### EXPLORE_AGENT Definition

Used in Phase 1 for parallel codebase discovery:

```typescript
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases',
  // All read-only tools: Glob, Grep, Read, Bash (read-only), WebFetch, WebSearch
  // Cannot: write files, edit files, spawn agents, exit plan mode
}
```

---

## 10. Exiting Plan Mode

**File:** `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`

### Tool Definition

```typescript
export const ExitPlanModeV2Tool: Tool = buildTool({
  name: 'ExitPlanMode',
  searchHint: 'present plan for approval and start coding (plan mode only)',
  shouldDefer: true,
  isConcurrencySafe: true,
  isReadOnly: false,  // writes plan file

  isEnabled() {
    if (getAllowedChannels().length > 0) return false
    return true
  },

  requiresUserInteraction() {
    // Teammates use mailbox approval (no local dialog)
    if (isTeammate()) return false
    return true  // standalone users require dialog
  },

  async validateInput(_input, { getAppState }) {
    const mode = getAppState().toolPermissionContext.mode
    if (mode !== 'plan') {
      return {
        result: false,
        message: 'You are not in plan mode. This tool is only for exiting plan mode.',
        errorCode: 1,
      }
    }
    return { result: true }
  },

  async checkPermissions(input, context) {
    if (isTeammate()) return { behavior: 'allow', updatedInput: input }
    return { behavior: 'ask', message: 'Exit plan mode?', updatedInput: input }
  },
})
```

### Plan Content Resolution

On `ExitPlanMode` call:
1. Check if `input.plan` was provided (set by CCR web UI edit)
2. Fall back to `getPlan(agentId)` — reads from disk
3. If CCR web UI provided plan → write it to disk (sync) + persist file snapshot

```typescript
const inputPlan = 'plan' in input && typeof input.plan === 'string'
  ? input.plan
  : undefined
const plan = inputPlan ?? getPlan(context.agentId)

if (inputPlan !== undefined && filePath) {
  await writeFile(filePath, inputPlan, 'utf-8')
  void persistFileSnapshotIfRemote()
}
```

### Mode Restoration

```typescript
context.setAppState(prev => {
  if (prev.toolPermissionContext.mode !== 'plan') return prev

  setHasExitedPlanMode(true)
  setNeedsPlanModeExitAttachment(true)

  let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'

  // Circuit breaker: if auto mode gate was disabled mid-plan, fall back to default
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
      restoreMode = 'default'
    }
  }

  return {
    ...prev,
    toolPermissionContext: {
      ...prev.toolPermissionContext,
      mode: restoreMode,
      prePlanMode: undefined,
    },
  }
})
```

### Tool Result Sent Back to Claude

**Standalone user approved:**
```
User has approved your plan. You can now start coding.

Your plan has been saved to: /path/to/plan.md

## Approved Plan:
{plan content}
```

**Sub-agent (plan agent):**
```
User has approved the plan. There is nothing else needed.
```

**Teammate awaiting leader:**
```
Your plan has been submitted to the team lead for approval.
Plan file: /path/to/plan.md
Request ID: plan_approval_...
```

### Input/Output Schema Stripping

`normalizeToolInput()` injects plan content from disk into the tool input before hooks/SDK see it:

```typescript
case EXIT_PLAN_MODE_V2_TOOL_NAME: {
  const plan = getPlan(agentId)
  const planFilePath = getPlanFilePath(agentId)
  void persistFileSnapshotIfRemote()
  return plan !== null ? { ...input, plan, planFilePath } : input
}
```

Before sending to the Anthropic API, `stripFields()` removes the injected fields (input schema expects an empty object):

```typescript
case EXIT_PLAN_MODE_V2_TOOL_NAME: {
  if (input && typeof input === 'object' &&
      ('plan' in input || 'planFilePath' in input)) {
    const { plan, planFilePath, ...rest } = input
    return rest
  }
  return input
}
```

---

## 11. Attachment System

**File:** `src/utils/attachments.ts`

Plan mode communicates its state to Claude via message attachments injected before API calls.

### Attachment Types

```typescript
| {
    type: 'plan_mode'
    reminderType: 'full' | 'sparse'
    isSubAgent?: boolean
    planFilePath: string
    planExists: boolean
  }
| {
    type: 'plan_mode_reentry'
    planFilePath: string
  }
| {
    type: 'plan_mode_exit'
    planFilePath: string
    planExists: boolean
  }
```

### Throttling Configuration

```typescript
export const PLAN_MODE_ATTACHMENT_CONFIG = {
  TURNS_BETWEEN_ATTACHMENTS: 5,           // Only attach every 5 turns
  FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,   // Full reminder every 5th attachment
} as const
```

### Attachment Injection Logic

```typescript
async function getPlanModeAttachments(
  messages: Message[] | undefined,
  toolUseContext: ToolUseContext,
): Promise<Attachment[]> {
  // Returns [] if mode !== 'plan'
  // Throttles: only every TURNS_BETWEEN_ATTACHMENTS turns
  // Adds 'plan_mode_reentry' if user re-entered plan in this session
  // Decides full vs sparse based on FULL_REMINDER_EVERY_N_ATTACHMENTS counter
  // Returns [{ type: 'plan_mode', reminderType: 'full' | 'sparse', ... }]
}

async function getPlanModeExitAttachment(
  toolUseContext: ToolUseContext,
): Promise<Attachment[]> {
  // Only sent once (one-time flag: needsPlanModeExitAttachment)
  // Returns [{ type: 'plan_mode_exit', planFilePath, planExists }]
  // Clears flag after sending
}
```

### What Each Attachment Contains

**`plan_mode` (full):** Complete workflow instructions (5-phase or interview loop), plan file path, enforcement constraints.

**`plan_mode` (sparse):** Brief reminder "You are in plan mode. Continue planning at {planFilePath}."

**`plan_mode_reentry`:** "You are returning to plan mode. Previous plan: {contents}."

**`plan_mode_exit`:** "You have exited plan mode. You can now make edits, run tools, and take actions. Reference: {planFilePath}."

---

## 12. Plan Mode State Machine

```
                        ┌─────────────────┐
                        │  default / auto  │
                        │   / dontAsk /    │
                        │  acceptEdits /   │
                        │ bypassPermissions│
                        └────────┬────────┘
                                 │
                    EnterPlanMode tool called
                                 │
                    prepareContextForPlanMode()
                    saves prePlanMode, strips dangerous perms
                                 │
                                 ▼
                        ┌────────────────┐
                        │      plan      │◄──── plan_mode attachment (every 5 turns)
                        │ (read-only     │
                        │  phase)        │
                        └────────┬───────┘
                                 │
                   ExitPlanMode tool called + user approves
                                 │
                   validateInput: must be in 'plan' mode
                   checkPermissions: 'ask' dialog
                                 │
                   handlePlanModeTransition('plan', prePlanMode)
                   sets needsPlanModeExitAttachment = true
                                 │
                                 ▼
                   ┌─────────────────────────────┐
                   │  restored prePlanMode        │
                   │  (or 'default' if fallback)  │◄──── plan_mode_exit attachment (once)
                   └─────────────────────────────┘
```

### Re-entry Detection

```typescript
// State tracking in src/bootstrap/state.ts
hasExitedPlanModeInSession: boolean    // true once any plan mode exit happens
needsPlanModeExitAttachment: boolean   // triggers one-time exit notification
```

If Claude enters plan mode again after having already exited in the same session, the `plan_mode_reentry` attachment is sent to provide context.

---

## 13. Team Coordination & Plan Approval

**File:** `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`, `src/utils/teammate.ts`

### Mandatory Plan Mode for Teammates

```typescript
export function isPlanModeRequired(): boolean {
  // Priority: in-process context > dynamic context > env var
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.planModeRequired
  if (dynamicTeamContext !== null) return dynamicTeamContext.planModeRequired
  return isEnvTruthy(process.env.CLAUDE_CODE_PLAN_MODE_REQUIRED)
}
```

CLI flag: `--plan-mode-required` (hidden from help). Used when spawning teammate agents.

### Approval Request Flow

When a teammate calls `ExitPlanMode` and `isPlanModeRequired()` is true:

```typescript
const approvalRequest = {
  type: 'plan_approval_request',
  from: getAgentName(),
  timestamp: new Date().toISOString(),
  planFilePath: filePath,
  planContent: plan,
  requestId: generateRequestId('plan_approval', formatAgentId(...)),
}

await writeToMailbox('team-lead', {
  from: getAgentName(),
  text: jsonStringify(approvalRequest),
  timestamp: new Date().toISOString(),
}, teamName)

// Update task state: awaiting_plan_approval = true
setAwaitingPlanApproval(agentTaskId, context.setAppState, true)
```

### Approval Response Display

**File:** `src/components/messages/PlanApprovalMessage.tsx`

```typescript
// Approval (green border):
"✓ Plan Approved by {senderName}"
"You can now proceed with implementation."

// Rejection (red border):
"✗ Plan Rejected by {senderName}"
"Feedback: {feedback text}"
"Please revise your plan based on the feedback and call ExitPlanMode again."
```

---

## 14. Feature Flags & Experiments

| Flag | File | Effect |
|------|------|--------|
| `tengu_plan_mode_interview_phase` | `planModeV2.ts` | Enables iterative interview workflow (always on for ants) |
| `tengu_pewter_ledger` | `planModeV2.ts` | Controls Phase 4 plan file size guidance (arms: null/trim/cut/cap) |
| `TRANSCRIPT_CLASSIFIER` | `permissionSetup.ts` | Enables auto mode interaction with plan (dangerous permission stripping) |
| `KAIROS` / `KAIROS_CHANNELS` | `EnterPlanModeTool` | When `--channels` is active, plan mode disabled |

### Pewter Ledger Experiment Detail

**Goal:** Reduce plan file size to lower API rejection rate  
**Background:** Rejection rate climbs from ~20% at <2K chars to ~50% at >20K chars  
**Baseline:** p50 = 4,906 chars, p90 = 11,617 chars  
**Arms:**

| Arm | Guidance |
|-----|---------|
| `null` (control) | No size guidance |
| `trim` | Light suggestion to be concise |
| `cut` | Moderate size constraints |
| `cap` | Strict maximum length guidance |

---

## 15. Agent Count by Subscription Tier

**File:** `src/utils/planModeV2.ts`

```typescript
export function getPlanModeV2AgentCount(): number {
  // Env override (max 10)
  if (process.env.CLAUDE_CODE_PLAN_V2_AGENT_COUNT) {
    const count = parseInt(process.env.CLAUDE_CODE_PLAN_V2_AGENT_COUNT, 10)
    if (!isNaN(count) && count > 0 && count <= 10) return count
  }

  const subscriptionType = getSubscriptionType()
  const rateLimitTier = getRateLimitTier()

  if (subscriptionType === 'max' && rateLimitTier === 'default_claude_max_20x') return 3
  if (subscriptionType === 'enterprise' || subscriptionType === 'team') return 3
  return 1  // free / standard
}

export function getPlanModeV2ExploreAgentCount(): number {
  // Env override (max 10)
  if (process.env.CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT) {
    const count = parseInt(process.env.CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT, 10)
    if (!isNaN(count) && count > 0 && count <= 10) return count
  }
  return 3  // always 3 explore agents by default
}
```

| Subscription | Plan Agents | Explore Agents |
|-------------|------------|---------------|
| Free / Standard | 1 | 3 |
| Max (20× tier) | 3 | 3 |
| Enterprise / Team | 3 | 3 |
| Env var override | 1–10 | 1–10 |

---

## 16. CLI Activation

### Slash Command

```
/plan           → Enter plan mode (or show current plan if already in plan mode)
/plan open      → Open plan file in external editor ($EDITOR)
```

### Keyboard Shortcut

**Shift+Tab** — Cycles through permission modes, including `plan`.

### CLI Flag

```
--plan-mode-required   (hidden from --help)
```

Forces spawned teammates to enter plan mode before implementation. Does not directly enter plan mode in the current session.

---

## 17. UI Components

**File:** `src/components/messages/PlanApprovalMessage.tsx`

### Plan Mode Active Indicator

Terminal UI shows current mode in the status bar. When `mode === 'plan'`, the color token `planMode` (blue/cyan) is used for:
- Border color of the plan approval request card
- Mode indicator in the prompt

### ExitPlanMode Dialog

The `checkPermissions` returning `{ behavior: 'ask' }` triggers the standard permission dialog:

```
┌─────────────────────────────────────────────┐
│  Exit plan mode?                            │
│                                             │
│  [Allow]  [Deny]                            │
└─────────────────────────────────────────────┘
```

### Plan Approval Cards (Teammates)

```
╭─ Plan Approval Request from {agentName} ────╮   (planMode colored border)
│                                              │
│  {plan content in markdown}                  │
│                                              │
│  Plan file: /path/to/plan.md                 │
╰──────────────────────────────────────────────╯

╭─ ✓ Plan Approved by {senderName} ───────────╮   (green border)
│  You can now proceed with implementation.    │
╰──────────────────────────────────────────────╯

╭─ ✗ Plan Rejected by {senderName} ───────────╮   (red border)
│  Feedback: {feedback text}                   │
│  Please revise and call ExitPlanMode again.  │
╰──────────────────────────────────────────────╯
```

---

## 18. Session Persistence & Recovery

**File:** `src/utils/plans.ts`

### Plan File Recovery on Session Resume

```typescript
export async function copyPlanForResume(
  log: LogOption,
  targetSessionId?: SessionId,
): Promise<boolean> {
  const slug = getSlugFromLog(log)
  if (!slug) return false

  const planPath = join(getPlansDirectory(), `${slug}.md`)

  try {
    await readFile(planPath, { encoding: 'utf-8' })
    return true  // file exists, nothing to do
  } catch (e) {
    if (!isENOENT(e)) return false

    // Recovery source 1: CCR file snapshot system message
    const snapshotPlan = findFileSnapshotEntry(log.messages, 'plan')
    if (snapshotPlan?.content.length > 0) {
      await writeFile(planPath, snapshotPlan.content, 'utf-8')
      return true
    }

    // Recovery source 2: ExitPlanMode tool_use input (plan field)
    // Recovery source 3: User message planContent field
    // Recovery source 4: plan_file_reference attachment
    const recovered = recoverPlanFromMessages(log)
    if (recovered) {
      await writeFile(planPath, recovered, 'utf-8')
      return true
    }
  }
  return false
}
```

Recovery priority:
1. Direct file read (fastest path)
2. CCR file snapshot in message history (for Cloudflare Workers pod recycling)
3. `ExitPlanMode` tool_use `input.plan` field
4. User message `planContent` field
5. `plan_file_reference` attachment

### Plan File Forking

When a session is forked:

```typescript
export async function copyPlanForFork(
  log: LogOption,
  targetSessionId: SessionId,
): Promise<boolean> {
  const originalSlug = getSlugFromLog(log)
  if (!originalSlug) return false

  const newSlug = getPlanSlug(targetSessionId)  // new unique slug
  await copyFile(
    join(plansDir, `${originalSlug}.md`),
    join(plansDir, `${newSlug}.md`),
  )
  return true
}
```

### CCR Remote Session Persistence

For remote (CCR) sessions where the plan file lives in a transient container:

```typescript
void persistFileSnapshotIfRemote()
```

Called when:
- `ExitPlanMode` is called
- CCR web UI provides edited plan content

Saves plan file contents as a snapshot message in the session history, ensuring recovery after pod recycling.

---

## 19. handlePlanModeTransition

**File:** `src/bootstrap/state.ts`

```typescript
export function handlePlanModeTransition(
  fromMode: string,
  toMode: string,
): void {
  // Entering plan mode: clear any pending exit attachment
  // (prevents sending plan_mode + plan_mode_exit in the same turn during rapid toggle)
  if (toMode === 'plan' && fromMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = false
  }

  // Leaving plan mode: trigger the plan_mode_exit attachment
  if (fromMode === 'plan' && toMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = true
  }
}
```

**Telemetry emitted** from `ExitPlanModeV2Tool`:

```typescript
logEvent('tengu_exit_plan_mode_called_outside_plan', {
  model: options.mainLoopModel,
  mode: mode,                                  // mode at time of call
  hasExitedPlanModeInSession: hasExitedPlanModeInSession(),
})
```

---

## 20. Key Constants Reference

| Constant | Value | File | Meaning |
|----------|-------|------|---------|
| `TURNS_BETWEEN_ATTACHMENTS` | 5 | `attachments.ts` | Plan mode reminder throttle interval |
| `FULL_REMINDER_EVERY_N_ATTACHMENTS` | 5 | `attachments.ts` | Full vs sparse reminder cycle |
| `MAX_WORKTREE_SLUG_LENGTH` | 64 | `worktree.ts` | Max length of generated word slug |
| `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` | env var | `planModeV2.ts` | Override plan agent count (1–10) |
| `CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT` | env var | `planModeV2.ts` | Override explore agent count (1–10) |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | env var | `teammate.ts` | Force plan-mode-required for teammates |
| `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` | env var | `planModeV2.ts` | Force interview phase on |
| Plan agent max nesting | 1 | `planAgent.ts` | Plan agents cannot spawn sub-agents |

---

## 21. File Inventory

| File | Purpose |
|------|---------|
| `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` | Tool: enter plan mode |
| `src/tools/EnterPlanModeTool/prompt.ts` | When-to-use guidance for EnterPlanMode |
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | Tool: exit plan mode, save plan, restore permissions |
| `src/utils/planModeV2.ts` | Interview phase gate, agent counts, pewter ledger |
| `src/utils/plans.ts` | Plan file path/slug/read/write/recovery/fork |
| `src/utils/attachments.ts` | plan_mode / plan_mode_exit / plan_mode_reentry attachment injection |
| `src/utils/permissions/permissionSetup.ts` | prepareContextForPlanMode, dangerous rule stripping |
| `src/types/permissions.ts` | PermissionMode union type definitions |
| `src/bootstrap/state.ts` | handlePlanModeTransition, needsPlanModeExitAttachment flag |
| `src/utils/messages.ts` | getPlanModeV2Instructions, getPlanModeInterviewInstructions |
| `src/tools/AgentTool/built-in/planAgent.ts` | PLAN_AGENT definition + system prompt |
| `src/tools/AgentTool/built-in/exploreAgent.ts` | EXPLORE_AGENT definition |
| `src/utils/teammate.ts` | isPlanModeRequired, teammate plan approval flow |
| `src/components/messages/PlanApprovalMessage.tsx` | Plan approval request/response UI cards |

---

## 22. Security Findings

### Strengths

| # | Finding | Detail |
|---|---------|--------|
| S1 | Explicit permission mode type system | `'plan'` is a first-class value in `PermissionMode`; cannot be confused with other modes |
| S2 | `prePlanMode` saved + restored | Mode accurately restored on exit, including dangerous-permission restoration |
| S3 | Circuit breaker on exit | If `auto` mode gate was disabled mid-plan, exit falls back to `'default'` instead of restoring to risky `auto` |
| S4 | `validateInput` guards ExitPlanMode | Returns error if not in plan mode — prevents spurious exit calls |
| S5 | `disallowedTools` on plan agents | Explicit blocklist prevents plan agents from editing files or spawning sub-agents |
| S6 | Path traversal defense in plans directory | `resolve()` check prevents `plansDirectory` config from escaping the project root |
| S7 | Plan file slug is random | Slug generated via `generateWordSlug()` — not user-controlled, no injection |
| S8 | Mandatory approval for teammates | `isPlanModeRequired()` + mailbox approval prevents autonomous code changes in multi-agent teams |
| S9 | Attachment throttling | Every-5-turns throttle prevents token waste from repeated full workflow instructions |

### Concerns

| # | Severity | Finding | Detail |
|---|----------|---------|--------|
| C1 | MEDIUM | No cryptographic verification of plan file | Plan file at `~/.claude/plans/{slug}.md` is a plain text file — any process with filesystem access can modify it between Claude writing it and ExitPlanMode reading it |
| C2 | MEDIUM | `CLAUDE_CODE_PLAN_V2_AGENT_COUNT=10` allows 10 parallel agents | Env var override allows up to 10 plan agents regardless of subscription tier — potential resource abuse |
| C3 | MEDIUM | CCR web UI can inject arbitrary plan content | `input.plan` from the web UI is written to disk without sanitization. If the web UI is compromised, malicious plan content could guide implementation |
| C4 | LOW | Plan file recovery trusts message history | `recoverPlanFromMessages()` reads plan content from past tool_use inputs and user messages — a specially crafted session could poison the recovered plan |
| C5 | LOW | `persistFileSnapshotIfRemote()` is fire-and-forget | Snapshot failures are silently swallowed; plan may not survive pod recycling in CCR |
| C6 | LOW | Interview phase always enabled for `ant` users | Anthropic employees always get the interview phase regardless of external gate state — behavior difference that could mask issues in user-facing path |
| C7 | INFO | Pewter ledger experiment can silently alter plan quality | Arms `cut` and `cap` may produce shorter, less complete plans without the user knowing the experiment is active |
| C8 | INFO | Rapid toggle edge case | If user enters and immediately exits plan mode, `handlePlanModeTransition` clears `needsPlanModeExitAttachment` — the model never receives the exit notification |
