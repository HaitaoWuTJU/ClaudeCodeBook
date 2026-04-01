# Summary of `tools/TaskStopTool/`

## Purpose of `TaskStopTool/`

Provides a tool for stopping running background tasks by their ID, serving as the modern replacement for the deprecated `KillShell` tool.

## Contents Overview

| File | Role | Key Responsibility |
|------|------|---------------------|
| `prompt.ts` | Configuration | Exports tool name (`TaskStopTool`) and description text |
| `TaskStopTool.ts` | Core Logic | Validates input, looks up task, calls `stopTask()`, returns structured output |
| `UI.tsx` | Presentation | Renders truncated command info with "· stopped" status in terminal UI |

## How Files Relate to Each Other

```
prompt.ts (metadata)
    │
    ├── exports TASK_STOP_TOOL_NAME, DESCRIPTION
    │
    ▼
TaskStopTool.ts (main tool)
    ├── imports DESCRIPTION, TASK_STOP_TOOL_NAME from ./prompt.js
    ├── imports outputSchema (self-referential, lazy)
    │
    ├── exports TaskStopTool definition
    │   └── references inputSchema, outputSchema, validateInput, call
    │
    ├── call() returns Output type
    │
    ▼
UI.tsx (rendering)
    ├── imports Output type from ./TaskStopTool.js
    └── exports renderToolUseMessage(), renderToolResultMessage()
```

## Key Takeaways

1. **Backward Compatibility**: The tool maintains `KillShell` as an alias and accepts deprecated `shell_id` input for seamless migration.

2. **Strict Validation**: Input is validated for both existence and runtime status (`status === 'running'`) before attempting to stop.

3. **Terminal-Optimized Display**: Commands are truncated to 2 lines / 160 chars for compact display in terminal environments using Ink.

4. **Error Handling**: Returns distinct error codes (`1` = invalid input, `3` = not running) for clear error categorization.

5. **Concurrency Safety**: The tool is marked as concurrency-safe and supports deferred execution, enabling reliable parallel task management.
