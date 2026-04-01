# Summary of `services/extractMemories/`

## Purpose of `extractMemories/`

The `extractMemories/` directory implements an **automatic memory extraction system** that runs as a background agent at the end of each query loop. It analyzes recent conversation messages and persists important information to persistent memory files in the project's auto-memory directory, reducing redundant re-explanations in future sessions.

## Contents Overview

The directory contains two files:

| File | Role |
|------|------|
| `index.ts` | Core extraction engine — orchestrates forked agent execution, tool restrictions, state management, and analytics |
| `prompts.ts` | Prompt templates — builds the system prompt for the extraction subagent with two variants (auto-only vs. combined auto + team memory) |

## How Files Relate to Each Other

```
executeExtractMemories()           ←── called by print.ts after model responds
       │
       ▼
executeExtractMemoriesImpl()      ←── checks feature flags, remote mode, in-progress
       │
       ▼
runExtraction()                   ←── core logic: throttle, manifest scan, prompt build
       │
       ├──► scanMemoryFiles()     ──► memdir/   (reads existing memory files)
       │
       ├──► buildExtractAutoOnlyPrompt()   ──┐
       │    (or)                                 │  ←── prompts.ts
       └──► buildExtractCombinedPrompt()  ─────┘
                                                   │
                                                   ▼
                                     runForkedAgent()
                                                   │
                    ┌──────────────────────────────┤
                    │         Forked Agent         │
                    │  ┌────────────────────────┐  │
                    │  │  Restricted tools:    │  │
                    │  │  • Read, Grep, Glob   │  │
                    │  │  • Bash (read-only)   │  │
                    │  │  • Edit, Write        │  │
                    │  │    (memory dir only)  │  │
                    │  └────────────────────────┘  │
                    │  Prompt includes:             │
                    │  • 4-type memory taxonomy      │
                    │  • Turn budget strategy        │
                    │  • Existing memories manifest  │
                    │  • What NOT to save            │
                    │  • Optional MEMORY.md index    │
                    └──────────────────────────────┘
                                                   │
                                                   ▼
                              extractWrittenPaths() + analytics + append system message
```

**`prompts.ts`** is a pure function library — it takes parameters and returns formatted strings. **`index.ts`** is the orchestrator that calls `prompts.ts` functions, manages state via closure scoping, and coordinates the forked agent lifecycle.

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Trigger** | Runs after the main agent finishes (no more tool calls), before the next user input |
| **Agent pattern** | Forked agent shares prompt cache with parent; runs with a strict 5-turn cap and restricted tools |
| **Memory taxonomy** | Four types: Current Project, Codebase Knowledge, Decision Log, Lessons Learned |
| **Mutual exclusion** | If the main agent writes memories directly, the forked extraction skips that turn |
| **Coalescing** | Consecutive calls during an in-flight extraction are coalesced into a single trailing run, only processing new messages |
| **Throttling** | Extraction runs at most every N eligible turns (default 1, configurable via `tengu_bramble_lintel`) |
| **Scope** | Supports two modes: auto-only (single `memory/` directory) and combined (private `memory/` + team `team_memory/`), controlled by `TEAMMEM` feature flag |
| **Index management** | Optionally maintains a `MEMORY.md` index file with short pointers to individual memories (~150 chars/entry) |
| **Safety** | Team memories cannot contain secrets; Bash tool restricted to read-only; Edit/Write restricted to memory directory paths only |
