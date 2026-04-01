# Summary of `tools/TaskCreateTool/`

## Purpose of `TaskCreateTool/`

Provides a complete, self-contained tool for creating tasks within a task list system. This directory encapsulates the entire lifecycle of a task creation tool—from naming and documentation (`constants.ts`, `prompt.ts`) to implementation and error handling (`TaskCreateTool.ts`).

## Contents Overview

| File | Role | Exports |
|------|------|---------|
| `constants.ts` | Centralized naming | `TASK_CREATE_TOOL_NAME` |
| `prompt.ts` | Dynamic prompt generation | `DESCRIPTION`, `getPrompt()` |
| `TaskCreateTool.ts` | Full tool implementation | `TaskCreateTool` class |

## How Files Relate to Each Other

```
constants.ts
   │
   └──▶ TASK_CREATE_TOOL_NAME ──────────────────────┐
                                                   │
prompt.ts                                          │
   ├──▶ DESCRIPTION ───────────────────────────────┤
   │                                               │
   └──▶ getPrompt() ── (conditional teammate tips)  │
                    │                              │
                    ▼                              ▼
         TaskCreateTool.ts ◀────────────────────────┘
              │
              ├──▶ Uses TASK_CREATE_TOOL_NAME (naming)
              ├──▶ Uses DESCRIPTION (tool metadata)
              ├──▶ Uses getPrompt() (agent instructions)
              ├──▶ Validates input with Zod schema
              ├──▶ Creates task via tasks.js utilities
              ├──▶ Runs hooks via hooks.js utilities
              └──▶ Manages app state (expandedView)
```

**Build order / dependency graph:**
1. `constants.ts` — Leaf module (no imports)
2. `prompt.ts` — Leaf module (only reads config)
3. `TaskCreateTool.ts` — Composite module (imports from both above and utilities)

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Feature flag** | Tool is only registered when `isTodoV2Enabled()` returns `true` |
| **Concurrency** | Tool is marked concurrency-safe and uses deferred (non-blocking) execution |
| **Rollback safety** | If any task-created hook returns a `blockingError`, the newly created task is automatically deleted |
| **Schema validation** | Uses lazy Zod schemas for both input (`subject`, `description`, optional `activeForm`/`metadata`) and output (`id`, `subject`) |
| **UI integration** | Automatically expands the task list view (`expandedView: 'tasks'`) after successful creation |
| **Prompt adaptability** | Prompt includes teammate assignment tips only when agent swarms feature is enabled |
| **No external deps** | Entire directory has zero external library dependencies (only `zod` for schema validation) |

### `tools/TaskCreateTool/TaskCreateTool.ts`

## Purpose
Defines a `TaskCreateTool` that allows agents to create tasks in a task list, including input validation, hook execution, error handling with automatic rollback, and UI state management.

## Key Components

| Component | Description |
|-----------|-------------|
| `inputSchema` | Zod schema defining required `subject` and `description`, optional `activeForm` and `metadata` fields |
| `outputSchema` | Zod schema for returning created task's `id` and `subject` |
| `TaskCreateTool` | Main tool definition with `buildTool`, configured for deferred execution and concurrency safety |
| `call()` method | Core logic: creates task, executes hooks, handles rollback on blocking errors, manages app state |
| `mapToolResultToToolResultBlockParam()` | Formats success message as `"Task #{id} created successfully: {subject}"` |

## Data Flow

```
Input (subject, description, activeForm?, metadata?)
        │
        ▼
  Validate with inputSchema
        │
        ▼
  createTask() → generates taskId
        │
        ▼
  executeTaskCreatedHooks() → iterates async generator
        │
   ┌────┴────┐
   │ errors? │
   └────┬────┘
    Yes │ No
        ▼         ▼
 deleteTask()   context.setAppState({ expandedView: 'tasks' })
        │                     │
        ▼                     ▼
   throw Error          Return { task: { id, subject } }
```

## Dependencies

| Source | Imports |
|--------|---------|
| **External** | `zod/v4` — schema validation |
| **Intra-repo** | `../../Tool.js` — `buildTool`, `ToolDef` |
| | `../../utils/hooks.js` — `executeTaskCreatedHooks`, `getTaskCreatedHookMessage` |
| | `../../utils/lazySchema.js` — `lazySchema` |
| | `../../utils/tasks.js` — `createTask`, `deleteTask`, `getTaskListId`, `isTodoV2Enabled` |
| | `../../utils/teammate.js` — `getAgentName`, `getTeamName` |
| | `./constants.js` — `TASK_CREATE_TOOL_NAME` |
| | `./prompt.js` — `DESCRIPTION`, `getPrompt` |

## Notable Details

- **Enabled condition**: `isTodoV2Enabled()` must return `true`
- **Concurrency safety**: `isConcurrencySafe() = true`
- **Deferred execution**: `shouldDefer: true` (non-blocking)
- **Auto-expansion**: Automatically sets `expandedView: 'tasks'` in app state after creation
- **Hook rollback**: If any `blockingError` is returned from hooks, the newly created task is deleted before re-throwing the combined error messages
- **Lazy schemas**: Both `inputSchema` and `outputSchema` use `lazySchema()` for deferred evaluation
