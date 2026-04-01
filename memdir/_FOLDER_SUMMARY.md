# Summary of `memdir/`

## Purpose of `memdir/`

The `memdir/` directory implements a complete, file-based, persistent memory system for a Claude CLI session. It handles memory discovery, relevance selection, path management, security validation, and prompt construction — giving the model a "second brain" backed by flat files on disk.

## Contents Overview

The directory contains **11 files** across three functional layers:

| Layer | Files | Role |
|---|---|---|
| **Prompt Construction** | `memdir.ts`, `teamMemPrompts.ts` | Build instruction blocks that tell the model how to save, search, and use memories |
| **Path Management** | `paths.ts`, `teamMemPaths.ts`, `memoryScan.ts` | Resolve, validate, and sanitize all memory filesystem paths |
| **Utilities** | `findRelevantMemories.ts`, `memoryTypes.ts`, `memoryAge.ts`, `memoryShapeTelemetry.ts`, `index.ts`, `memdir.spec.ts` | Selection logic, type taxonomy, staleness warnings, analytics, and tests |

## How Files Relate to Each Other

```
paths.ts ──── getAutoMemPath()
  │              │
  ├──────────────┴─────► memdir.ts ── buildMemoryPrompt()
  │                               ── buildMemoryLines()
  │                               ── buildSearchingPastContextSection()
  │                                    (uses scanMemoryFiles from memoryScan.ts)
  │
  ├─ teamMemPaths.ts ──── getTeamMemPath()
  │     │                    │
  │     │                    ├──► teamMemPrompts.ts ── buildCombinedMemoryPrompt()
  │     │                    │         (combines auto + team prompts)
  │     │                    │
  │     └─ validateTeamMemWritePath()
  │        (called from callers writing team memory files)
  │
  └─ findRelevantMemories.ts
       │   │
       │   └─ scanMemoryFiles()  ──► reads MEMORY.md frontmatter from memoryScan.ts
       │                            calls sideQuery() with Sonnet model
       │                            optionally emits memoryShapeTelemetry()
       │
       └─ memoryAge.ts ── staleness warnings injected into prompts
```

**Call chain from the outside in:**

1. Bootstrap calls `systemPromptSection()` (imported from `memdir.ts` via `index.ts`) to produce the memory prompt string
2. `systemPromptSection()` calls `loadMemoryPrompt()` → dispatches to auto/team/combined/daily-log builders
3. The prompt builders read actual `MEMORY.md` files via `scanMemoryFiles()` (lazy — only when the file exists)
4. Before reading, `ensureMemoryDirExists()` is called to guarantee the directory exists
5. `findRelevantMemories()` is called during a session to select up to 5 memories relevant to the current query
6. `teamMemPaths.ts` validates all team memory write paths and key lookups before any file operation occurs

## Key Takeaways

### Two-Memory Architecture
The system maintains **two parallel memory directories**:
- **Auto-memory** (`<autoMemPath>/MEMORY.md`): Private, per-user, per-project
- **Team-memory** (`<autoMemPath>/team/MEMORY.md`): Shared across teammates

### Closed Taxonomy
All memories use a four-type taxonomy (user / feedback / project / reference) with explicit rules about what **not** to save (git state, code contents, file listings, file summaries). This keeps the index lightweight.

### LLM-Powered Selection
`findRelevantMemories.ts` uses a Sonnet model via `sideQuery()` with structured JSON output to pick the top 5 most relevant memories for a query — filtering out already-surfaced ones and, when tools are recently used, excluding tool reference docs (but keeping warnings/gotchas).

### Security-First Path Handling
`teamMemPaths.ts` implements **dual-pass symlink validation** for all team memory writes:
1. Fast string-level containment check (`path.startsWith(teamDir)`)
2. Filesystem-level symlink resolution with dangling-symlink and ELOOP detection

This defends against PSR M22186/22187 path traversal attacks where an attacker replaces a path component with a symlink.

### Feature-Gated Flexibility
Multiple GrowthBook feature flags control behavior: `tengu_coral_fern` (search section), `tengu_moth_copse` (skipIndex), `tengu_passport_quail` + `tengu_slate_thimble` (extract mode), `MEMORY_SHAPE_TELEMETRY` (analytics).

### Staleness as a First-Class Concept
`memoryAge.ts` and the `MEMORY_DRIFT_CAVEAT` constant make age and staleness explicit — the model is reminded that memories are point-in-time observations and is instructed to signal when retrieved memories are more than a day old.
