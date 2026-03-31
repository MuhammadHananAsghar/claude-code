# Claude Code Memory System — Deep Dive

> Based on source analysis of the leaked npm source map extraction (March 31, 2026).
> All type definitions, constants, and logic come directly from `src/memdir/`, `src/utils/`, `src/services/extractMemories/`, and related files.

---

## Table of Contents

1. [Overview](#overview)
2. [Memory Tiers](#memory-tiers)
3. [Tier 1 — CLAUDE.md Files (Static Instructions)](#tier-1--claudemd-files-static-instructions)
4. [Tier 2 — Auto Memory / memdir (Dynamic Persistent Memory)](#tier-2--auto-memory--memdir-dynamic-persistent-memory)
5. [Tier 3 — Team Memory (Collaborative)](#tier-3--team-memory-collaborative)
6. [Tier 4 — Agent Memory (Per-Agent Scope)](#tier-4--agent-memory-per-agent-scope)
7. [Memory Types Taxonomy](#memory-types-taxonomy)
8. [Storage Paths & Path Resolution](#storage-paths--path-resolution)
9. [MEMORY.md Index — Limits & Truncation](#memorymd-index--limits--truncation)
10. [Memory Scanning & Manifest](#memory-scanning--manifest)
11. [Relevant Memory Selection (AI-Powered Filtering)](#relevant-memory-selection-ai-powered-filtering)
12. [Memory Aging & Staleness](#memory-aging--staleness)
13. [Background Memory Extraction](#background-memory-extraction)
14. [Auto-Dream Consolidation](#auto-dream-consolidation)
15. [KAIROS Mode — Daily Log System](#kairos-mode--daily-log-system)
16. [Context Loading Flow](#context-loading-flow)
17. [Feature Flags](#feature-flags)
18. [Security & Path Validation](#security--path-validation)
19. [What NOT to Save](#what-not-to-save)
20. [Full Directory Structure](#full-directory-structure)

---

## Overview

Claude Code has a **multi-tier, file-based memory system** that persists information across conversations. It is not a vector database or embedding store — it is plain Markdown files with YAML frontmatter, read and written by the model itself using its standard file tools (Read, Write, Edit).

There are four distinct memory tiers:

| Tier | Name | Storage | Scope | Mutable by Claude? |
|------|------|---------|-------|-------------------|
| 1 | CLAUDE.md files | Various locations | Project/User/Global | No (static) |
| 2 | Auto Memory (memdir) | `~/.claude/projects/{project}/memory/` | Per-project, per-user | Yes |
| 3 | Team Memory | `~/.claude/projects/{project}/memory/team/` | Per-project, shared | Yes |
| 4 | Agent Memory | `~/.claude/agent-memory/{agent-type}/` | Per-agent-type | Yes |

All tiers are loaded into the system prompt before each conversation begins.

---

## Memory Tiers

### Tier 1 — CLAUDE.md Files (Static Instructions)

**Source**: `src/utils/claudemd.ts`

CLAUDE.md files are static, version-controlled instruction files. Claude reads them but does not modify them during conversation. They define project conventions, coding standards, behavioral rules, and any context the developer wants Claude to always have.

#### Discovery Hierarchy

Files are loaded in this order (later = higher priority, overrides earlier):

```
1.  /etc/claude-code/CLAUDE.md              ← admin-managed global (lowest priority)
2.  ~/.claude/CLAUDE.md                     ← user global
3.  ~/.claude/rules/**/*.md                 ← user rules directory
4.  <ancestor-dir>/CLAUDE.md                ← walking from cwd up to filesystem root
5.  <ancestor-dir>/.claude/CLAUDE.md        ← .claude variant
6.  <ancestor-dir>/.claude/rules/*.md       ← rules directory variant
7.  <cwd>/CLAUDE.local.md                   ← private project-local (highest priority)
```

The walk from cwd to root means a monorepo root `CLAUDE.md` is lower priority than a sub-package `CLAUDE.md` — closer files override further files.

#### Key Constants

```typescript
export const MAX_MEMORY_CHARACTER_COUNT = 40000  // per file
```

#### @include Directives

CLAUDE.md files support composing other files:

```markdown
@./relative/path.md
@~/home/relative/path.md
@/absolute/path.md
```

- Circular references are prevented via a `processedFiles` Set
- Non-existent includes are silently ignored
- Includes are resolved recursively

#### Frontmatter with Glob Filtering

Files can declare which file paths they apply to:

```yaml
---
paths:
  - "src/**/*.ts"
  - "tests/**"
---
Only inject this context when working with TypeScript source files.
```

#### Disabling CLAUDE.md

```bash
CLAUDE_CODE_DISABLE_CLAUDE_MDS=1  # env var disables all CLAUDE.md loading
--bare                             # CLI flag also disables (unless --add-dir specified)
```

---

### Tier 2 — Auto Memory / memdir (Dynamic Persistent Memory)

**Source**: `src/memdir/memdir.ts`, `src/memdir/paths.ts`

This is Claude's writable long-term memory. Claude saves memories here during or after conversations and retrieves them in future sessions.

#### How It Works

Two-step save process (enforced by the memory system prompt):

1. **Write a topic file** — e.g., `user_role.md`, `feedback_no_mocks.md`
2. **Add a pointer** to `MEMORY.md` index — one line, ~150 chars max

Example topic file:

```markdown
---
name: user prefers no database mocks
description: User wants integration tests hitting real DB, not mocks
type: feedback
---

Do not mock the database in tests.

**Why:** Prior incident where mock/prod divergence masked a broken migration.

**How to apply:** All test suites touching DB must use a real test database.
```

Example MEMORY.md pointer:

```markdown
- [No DB mocks](feedback_no_db_mocks.md) — integration tests must hit real database, never mock
```

#### Enabling/Disabling

```typescript
// src/memdir/paths.ts
export function isAutoMemoryEnabled(): boolean {
  // Disabled if:
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY)) return false
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) return false
  // Remote sessions without explicit memory dir also disabled:
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
      !process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) return false
  // Otherwise check settings.json autoMemoryEnabled (default: true)
}
```

---

### Tier 3 — Team Memory (Collaborative)

**Source**: `src/memdir/teamMemPaths.ts`, `src/memdir/teamMemPrompts.ts`

Optional shared memory that all team members working on the same project directory contribute to and read from.

- **Path**: `~/.claude/projects/{project}/memory/team/`
- **Separate MEMORY.md index** from personal memory
- **Synced** at the beginning of every session
- **Gated** by feature flag `tengu_herring_clock`
- **Security**: Path traversal, symlink escapes, null-byte injection all blocked (see [Security section](#security--path-validation))
- **Restriction**: Never save sensitive data (API keys, credentials) to team memory

When team memory is enabled, Claude has two directories — private (`autoDir`) and shared (`teamDir`) — and must decide which scope fits each memory.

---

### Tier 4 — Agent Memory (Per-Agent Scope)

**Source**: `src/tools/AgentTool/agentMemory.ts`

Built-in sub-agents (verification, plan, explore, guide, etc.) have their own persistent memory directories, separate from user memories.

#### Scopes

```typescript
export type AgentMemoryScope = 'user' | 'project' | 'local'
```

| Scope | Path | VCS? | Note |
|-------|------|------|------|
| `user` | `~/.claude/agent-memory/{agent-type}/` | No | Global across all projects |
| `project` | `.claude/agent-memory/{agent-type}/` | Yes | Shared with team via git |
| `local` | `.claude/agent-memory-local/{agent-type}/` | No | Machine-specific |

#### Scope Guidance in Prompts

- **user**: Keep learnings general — applies across all projects
- **project**: Tailor to this project — shared with team via VCS
- **local**: Tailor to this project and machine — not in VCS

---

## Memory Types Taxonomy

**Source**: `src/memdir/memoryTypes.ts`

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
export type MemoryType = (typeof MEMORY_TYPES)[number]
```

| Type | What to store | When to save | Body structure |
|------|--------------|--------------|----------------|
| `user` | Role, goals, expertise, preferences | When learning who the user is | Plain description |
| `feedback` | Behavioral guidance — what to avoid or repeat | On corrections OR validated non-obvious approaches | Rule → **Why:** → **How to apply:** |
| `project` | Non-derivable work context (deadlines, decisions, incidents) | When learning about work goals/state; convert relative dates to absolute | Fact → **Why:** → **How to apply:** |
| `reference` | External system pointers (Linear, Grafana, Slack) | When learning where info lives | Plain pointer with context |

### Frontmatter Format

```markdown
---
name: short descriptive name
description: one-line hook — used for relevance scoring in future sessions, be specific
type: user | feedback | project | reference
---

Memory content here.

**Why:** Reason this matters.
**How to apply:** When this should influence behavior.
```

---

## Storage Paths & Path Resolution

**Source**: `src/memdir/paths.ts`

### Auto Memory Path Resolution (Priority Order)

```
1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE env var   ← explicit override (Cowork/remote)
2. settings.json autoMemoryDirectory            ← trusted sources only
3. <memoryBase>/projects/<sanitized-git-root>/memory/
```

Where `memoryBase` = `~/.claude` (or `CLAUDE_CODE_REMOTE_MEMORY_DIR` if set).

The `sanitized-git-root` is the canonical git root path sanitized for use as a directory name, so all git worktrees of the same repo share memory.

### Daily Log Path (KAIROS mode)

```
~/.claude/projects/{project}/memory/logs/YYYY/MM/YYYY-MM-DD.md
```

### Key Path Functions

```typescript
getAutoMemPath()         // returns the memory directory path
getAutoMemEntrypoint()   // returns MEMORY.md path
getAutoMemDailyLogPath() // returns today's log path (KAIROS mode)
getTeamMemPath()         // returns team memory directory
getTeamMemEntrypoint()   // returns team MEMORY.md path
```

---

## MEMORY.md Index — Limits & Truncation

**Source**: `src/memdir/memdir.ts`

```typescript
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000  // ~25KB
```

When MEMORY.md exceeds either limit, it is **truncated** before being injected into context, and a warning is appended:

```
> WARNING: MEMORY.md is 247 lines (limit: 200). Only part of it was loaded.
> Keep index entries to one line under ~200 chars; move detail into topic files.
```

Each directory (personal + team in combined mode) has its own MEMORY.md with its own 200-line / 25KB limit.

---

## Memory Scanning & Manifest

**Source**: `src/memdir/memoryScan.ts`

At session start, Claude Code scans the memory directory for all `.md` files (excluding `MEMORY.md` itself).

```typescript
const MAX_MEMORY_FILES = 200       // maximum files scanned
const FRONTMATTER_MAX_LINES = 30   // only reads first 30 lines for headers
```

### Scan Process

1. `readdir(memoryDir, { recursive: true })` — find all `.md` files
2. For each file, read only the first 30 lines (for frontmatter)
3. Parse `description` and `type` from frontmatter
4. Sort by `mtime` descending (most recently modified first)
5. Truncate to `MAX_MEMORY_FILES = 200`

### Manifest Format

```
- [user] user_role.md (2026-03-31T10:00:00Z): user is a senior TypeScript engineer
- [feedback] no_db_mocks.md (2026-03-30T14:22:00Z): integration tests must hit real DB
- [project] auth_rewrite.md (2026-03-29T09:11:00Z): auth middleware rewrite due to legal compliance
```

---

## Relevant Memory Selection (AI-Powered Filtering)

**Source**: `src/memdir/findRelevantMemories.ts`

Rather than injecting all memories into every context, Claude Code uses **Claude Sonnet itself** to select the most relevant memories for the current query.

### Selection Process

1. Build manifest of all memory files (headers only)
2. Send manifest + user query to Claude Sonnet with selection prompt
3. Get back up to **5 filenames**
4. Load and inject only those files
5. Track `alreadySurfaced` to avoid re-surfacing files already shown this session

### Selection Prompt (actual system prompt from source)

> You are selecting memories that will be useful to Claude Code as it processes a user's query. You will be given the user's query and a list of available memory files with their filenames and descriptions.
>
> Return a list of filenames for the memories that will clearly be useful to Claude Code as it processes the user's query (up to 5). Only include memories that you are certain will be helpful based on their name and description.
> - If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
> - If there are no memories that would clearly be useful, feel free to return an empty list.
> - If a list of recently-used tools is provided, do not select memories that are usage reference or API documentation for those tools. **DO** still select memories containing warnings, gotchas, or known issues about those tools — active use is exactly when those matter.

### Key Behavior

- **Max 5 memories** per query
- **Skips already-surfaced** memories within a session
- **Tool-aware**: won't re-surface API docs for tools already in use, but **will** surface gotchas/warnings for those same tools
- Runs a small, fast Sonnet call — not the full model

---

## Memory Aging & Staleness

**Source**: `src/memdir/memoryAge.ts`

Memories are point-in-time snapshots. The system tracks `mtime` and attaches freshness caveats to old memories.

```typescript
export function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}

export function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}
```

### Freshness Caveat (injected for memories > 1 day old)

```
<system-reminder>
This memory is 47 days old. Memories are point-in-time observations, not live state —
claims about code behavior or file:line citations may be outdated.
Verify against current code before asserting as fact.
</system-reminder>
```

This is wrapped in `<system-reminder>` tags so it appears as system context rather than conversation content.

---

## Background Memory Extraction

**Source**: `src/services/extractMemories/extractMemories.ts`

At the end of each query loop (when the model produces a final response with no remaining tool calls), a **background forked agent** automatically extracts durable memories from the conversation transcript.

### Architecture

- **Perfect fork**: shares the parent conversation's prompt cache (cost-efficient)
- **Closure-scoped state**: cursor tracking, overlap prevention, coalescing of concurrent calls
- **Trigger**: model produces final response → extraction fires asynchronously
- **Non-blocking**: main conversation is not held up waiting for extraction

### Tool Permissions for Extraction Agent

The extraction agent has limited tool access — it can only:

```typescript
// Allowed unrestricted:
FILE_READ_TOOL_NAME  // Read
GREP_TOOL_NAME       // Grep
GLOB_TOOL_NAME       // Glob

// Allowed if read-only command:
BASH_TOOL_NAME       // ls, find, grep, cat, stat, wc, head, tail

// Allowed ONLY within memory directory:
FILE_EDIT_TOOL_NAME  // Edit (memory files only)
FILE_WRITE_TOOL_NAME // Write (memory files only)

// Everything else: DENIED
```

This prevents the extraction agent from modifying project code or running arbitrary commands.

### Feature Gate

```
tengu_passport_quail   ← enables background extraction
tengu_slate_thimble    ← enables extraction in non-interactive sessions
```

### Coalescing

If an extraction is already in-progress when another conversation ends, the new context is queued as `pendingContext` and processed after the current one completes — preventing overlapping extractions.

---

## Auto-Dream Consolidation

**Source**: `src/services/autoDream/autoDream.ts`

A background consolidation service that periodically distills accumulated memories and session logs into a clean, organized MEMORY.md and topic files.

### Trigger Conditions (both must pass)

```typescript
type AutoDreamConfig = {
  minHours: number    // default: 24 — hours since last consolidation
  minSessions: number // default: 5  — sessions accumulated since last run
}
```

- Time gate: `hoursSinceLastConsolidation >= minHours`
- Session gate: `accumulatedSessions >= minSessions`
- Lock: prevents overlapping consolidations
- Scan throttle: `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` (10 min)

### Enable Check

```typescript
export function isAutoDreamEnabled(): boolean {
  const setting = getInitialSettings().autoDreamEnabled
  if (setting !== undefined) return setting
  // Falls back to growthbook feature flag: tengu_onyx_plover
}
```

---

## KAIROS Mode — Daily Log System

**Source**: `src/memdir/memdir.ts`, `src/bootstrap/state.ts`

An alternative memory mode for **long-lived assistant sessions** (as opposed to short developer sessions). Instead of the standard MEMORY.md + topic files pattern, KAIROS uses append-only daily logs.

### How It Works

1. Claude appends timestamped bullet entries to today's log file during the session
2. A separate nightly `/dream` skill distills the logs into topic files + MEMORY.md
3. MEMORY.md is still loaded as the distilled index

### Log Path

```
~/.claude/projects/{project}/memory/logs/YYYY/MM/YYYY-MM-DD.md
```

When the date rolls over mid-session, Claude starts appending to the new day's file.

### Activation

```typescript
// State flag
export function getKairosActive(): boolean { return state.kairosActive }
export function setKairosActive(value: boolean): void { state.kairosActive = value }

// Feature flag
feature('KAIROS')  // compile-time bun:bundle feature
```

When `feature('KAIROS') && isAutoMemoryEnabled() && getKairosActive()` → use daily log mode instead of standard memdir mode.

---

## Context Loading Flow

**Source**: `src/context.ts`

```
getUserContext()
    │
    ├── shouldDisableClaudeMd?
    │   └── CLAUDE_CODE_DISABLE_CLAUDE_MDS env var
    │   └── isBareMode() && no --add-dir
    │
    ├── getMemoryFiles()
    │   └── loads CLAUDE.md hierarchy files
    │   └── loads auto memory MEMORY.md + relevant topic files
    │   └── loads team MEMORY.md + relevant topic files (if team enabled)
    │
    ├── filterInjectedMemoryFiles()
    │   └── deduplication + security filtering
    │
    └── getClaudeMds(filteredFiles)
        └── assembles final claudeMd string
        └── setCachedClaudeMdContent() ← cached for yoloClassifier

Return: { claudeMd: "...", currentDate: "Today's date is ..." }
```

The assembled `claudeMd` string is injected into the system prompt before every conversation.

---

## Feature Flags

Memory behavior is controlled by Growthbook feature flags (runtime) and Bun bundle features (compile-time).

### Runtime Feature Flags (Growthbook)

| Flag | Effect |
|------|--------|
| `tengu_herring_clock` | Enable team memory |
| `tengu_moth_copse` | Skip MEMORY.md index (still load topic files) |
| `tengu_coral_fern` | Enable "Searching past context" section in memory prompt |
| `tengu_passport_quail` | Enable background memory extraction agent |
| `tengu_slate_thimble` | Enable extraction in non-interactive sessions |
| `tengu_onyx_plover` | Enable auto-dream consolidation |
| `MEMORY_SHAPE_TELEMETRY` | Enable memory recall analytics |

### Compile-Time Bundle Features (Bun)

| Feature | Effect |
|---------|--------|
| `KAIROS` | Enable assistant-mode daily log memory |
| `TEAMMEM` | Enable team memory system code paths |
| `EXTRACT_MEMORIES` | Enable memory extraction system |
| `AGENT_MEMORY_SNAPSHOT` | Include agent memory snapshot in sub-agents |

---

## Security & Path Validation

**Source**: `src/memdir/teamMemPaths.ts`

Team memory has strict path validation to prevent malicious memory files from escaping the memory directory.

### Protections

```typescript
export class PathTraversalError extends Error { ... }

export async function validateTeamMemWritePath(filePath: string): Promise<string> {
  // 1. Null byte check
  if (filePath.includes('\0')) throw new PathTraversalError(...)

  // 2. Path must stay within team directory
  const resolvedPath = resolve(filePath)
  if (!resolvedPath.startsWith(teamDir)) throw new PathTraversalError(...)

  // 3. Symlink traversal check (realpathDeepestExisting)
  const realPath = await realpathDeepestExisting(resolvedPath)
  if (!(await isRealPathWithinTeamDir(realPath))) throw new PathTraversalError(...)

  return resolvedPath
}
```

Three-layer defense:
1. **Null bytes** — blocked outright
2. **Path traversal** (`../../../etc/passwd`) — `resolve()` + prefix check
3. **Symlink escapes** — `realpathDeepestExisting()` resolves symlinks before checking prefix

Personal memory does not have the same validation (it's user-scoped), but auto memory file detection (`isAutoMemPath`) is used to restrict what the extraction agent can write to.

---

## What NOT to Save

The memory system prompt explicitly instructs Claude not to save:

- **Code patterns, conventions, architecture, file paths** — derivable by reading current code
- **Git history, recent changes, who-changed-what** — `git log`/`git blame` are authoritative
- **Debugging solutions or fix recipes** — the fix is in the code; the commit has the context
- **Anything already documented in CLAUDE.md files**
- **Ephemeral task details** — in-progress work, current session context

The rationale: memory is for things that are **non-obvious** and **non-derivable**. Saving derivable state creates drift and false confidence.

---

## Full Directory Structure

```
~/.claude/
│
├── CLAUDE.md                          ← user global static instructions
│
└── projects/
    └── {sanitized-git-root}/
        └── memory/                    ← auto memory root (Tier 2)
            │
            ├── MEMORY.md              ← index (max 200 lines / 25KB)
            │
            ├── user_*.md              ← user type memories
            ├── feedback_*.md          ← feedback type memories
            ├── project_*.md           ← project type memories
            ├── reference_*.md         ← reference type memories
            │
            ├── logs/                  ← KAIROS daily logs
            │   └── YYYY/
            │       └── MM/
            │           └── YYYY-MM-DD.md
            │
            └── team/                  ← team memory root (Tier 3, optional)
                └── MEMORY.md          ← shared team index

~/.claude/
└── agent-memory/
    └── {agent-type}/                  ← user-scoped agent memory (Tier 4)
        ├── MEMORY.md
        └── *.md

{project}/
└── .claude/
    ├── agent-memory/
    │   └── {agent-type}/              ← project-scoped agent memory (Tier 4)
    └── agent-memory-local/
        └── {agent-type}/              ← local-scoped agent memory (Tier 4)
```

---

## Key Insights for Security Research

1. **Memory files are plain Markdown** — no encryption, stored in `~/.claude/projects/`
2. **The extraction agent has restricted tool access** — cannot write outside memory directory
3. **Team memory has path traversal protection** — three-layer defense (null bytes, path resolution, symlink resolution)
4. **Personal memory has no write restrictions beyond isAutoMemPath check** — the extraction agent relies on path prefix matching
5. **Memory is loaded into system prompt verbatim** — a malicious CLAUDE.md or memory file that gets loaded could influence model behavior (prompt injection vector)
6. **The AI-powered relevance filter uses Claude Sonnet** — a separate API call happens per session when memories exist; this is a potential cost amplification point
7. **MEMORY.md is truncated at 200 lines** — old entries silently dropped; could be exploited to push out important safety memories by flooding the index
8. **`autoMemoryDirectory` in settings.json is honored** — if settings can be modified, memory path can be redirected
