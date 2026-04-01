# Summary of `tools/TaskUpdateTool/`

## Purpose of `TaskUpdateTool/`

This directory implements the `TaskUpdateTool` — a Todo V2 system tool that allows AI agents to modify tasks. It provides comprehensive task update capabilities including status transitions, field modifications, ownership management, blocking relationships, metadata updates, and task deletion.

## Contents Overview

| File | Role |
|------|------|
| `constants.ts` | Defines the canonical tool name string (`'TaskUpdate'`) to avoid hardcoding literals throughout the codebase |
| `prompt.ts` | Exports `DESCRIPTION` and `PROMPT` — human-readable instructions guiding the AI agent on when and how to use the tool, including completion criteria, updatable fields, and JSON examples |
| `TaskUpdateTool.ts` | **Main implementation** — defines the tool's input/output Zod schemas and the `call()` execution logic, integrating with the task CRUD layer, hook system, mailbox notifications, and feature flags |

## How Files Relate to Each Other

```
constants.ts
    └── TASK_UPDATE_TOOL_NAME (string)
            ↓
    used by TaskUpdateTool.ts (tool definition name)

prompt.ts
    ├── DESCRIPTION ────────────────────────┐
    └── PROMPT ────────────────────────────┴──→ TaskUpdateTool.ts
                                                    (passed to buildTool as metadata)
```

1. **`constants.ts`** exports the tool name constant — consumed by external callers (e.g., tool registration/dispatch logic)
2. **`prompt.ts`** exports the description and instruction prompt strings — passed into `buildTool()` to configure the AI agent's behavior
3. **`TaskUpdateTool.ts`** imports metadata from both files, defines the operational schemas and logic, and is the single entry point that ties everything together

## Key Takeaways

- **Status as action** — The special `'deleted'` status literal bypasses normal updates and triggers `deleteTask()` directly
- **Auto-ownership** — Moving a task to `in_progress` without an explicit owner auto-assigns it to the calling agent, enabling activity tracking
- **Completion hooks** — Marking a task `completed` triggers a hook system; if any hook reports a blocking error, the update fails
- **Non-error not-found** — When a task doesn't exist, the tool returns a non-error result to avoid sibling tool cancellation in streaming execution
- **Verification nudge** — On long task lists where all tasks are completed, a nudge flag is raised to suggest creating a verification task
- **Blocking model** — The tool maintains bidirectional blocking relationships (`addBlocks` / `addBlockedBy`) via `blockTask()`
- **Feature-gated behavior** — Verification nudge and agent swarms mailbox notifications are gated behind GrowthBook feature flags
