# Summary of `tools/TodoWriteTool/`

## Purpose of `TodoWriteTool/`

Provides a session-based todo list management tool for AI coding assistants. It enables agents to create, update, track, and persist task checklists during coding sessions, with built-in verification nudge logic to encourage users to add verification steps after completing multiple tasks.

## Contents Overview

| File | Purpose | Size |
|------|---------|------|
| `constants.ts` | Exports the tool's canonical name constant | ~50 bytes |
| `prompt.ts` | Defines system prompt template and description for tool usage guidelines | ~9KB |
| `TodoWriteTool.ts` | Core tool implementation with schema validation, execution logic, and output formatting | ~4KB |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                      TodoWriteTool.ts                           │
│                                                                 │
│  • Imports: TODO_WRITE_TOOL_NAME, DESCRIPTION, PROMPT           │
│  • Uses them to configure the tool definition                   │
│  • Exports: TodoWriteTool, isEnabled(), mapToolResultTo...()   │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ imports
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│  constants.ts   │  │   prompt.ts     │  │  (external deps)    │
│                 │  │                 │  │                     │
│ • Provides the  │  │ • PROMPT (~9KB)  │  │ • zod/v4 schemas    │
│   tool name     │  │   with rules &  │  │ • appState (read)   │
│                 │  │   examples      │  │ • feature flags     │
│ • Single        │  │                 │  │                     │
│   constant      │  │ • DESCRIPTION   │  │                     │
│   export        │  │   short text    │  │                     │
└─────────────────┘  └─────────────────┘  └─────────────────────┘
```

**Data flow within directory:**
1. `constants.ts` → minimal string export (`'TodoWrite'`)
2. `prompt.ts` → detailed instructions + metadata
3. `TodoWriteTool.ts` → orchestrates everything; imports from both files to build the complete tool

## Key Takeaways

1. **Single-responsibility modules** — Each file has one clear job: name, documentation, or implementation

2. **Feature-gated** — Tool disables itself when `isTodoV2Enabled()` is true, indicating a migration path to a newer implementation

3. **Verification nudge system** — Unique feature that detects when 3+ tasks are completed without any verification steps and prompts the agent to add verification

4. **State persistence** — Uses `appState.todos[todoKey]` with agent/session-scoped keys for multi-agent support

5. **Schema-validated I/O** — Input/output validation via Zod v4 ensures type safety and data integrity

6. **DRY principle** — Tool name centralized in `constants.ts` to avoid hardcoding
