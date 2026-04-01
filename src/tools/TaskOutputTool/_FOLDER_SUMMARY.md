# Summary of `tools/TaskOutputTool/`

## Purpose of `TaskOutputTool/`

This directory contains the implementation of the `TaskOutputTool` — a specialized tool designed to retrieve and display output from background tasks (shell commands, local agents, and remote agents) within the Continue development environment.

## Contents Overview

The directory contains a single file that implements the complete tool functionality:

| File | Description |
|------|-------------|
| `TaskOutputTool.tsx` | Main implementation with tool definition, input validation, execution logic, result rendering, and TypeScript types |
| `constants.ts` | Shared constant `TASK_OUTPUT_TOOL_NAME` used as the tool identifier |

## How Files Relate to Each Other

```
constants.ts
    │
    └──► Exports TASK_OUTPUT_TOOL_NAME constant
              │
              └──► Imported by TaskOutputTool.tsx
                        │
                        └──► Used to register the tool with its standardized name
```

The relationship is straightforward: `constants.ts` provides a centralized string constant that `TaskOutputTool.tsx` imports and uses when defining the tool. This avoids hardcoding the tool name string in multiple places.

## Key Takeaways

1. **Blocking & Non-blocking Modes**: The tool supports retrieving task output either immediately (`block=false`) or waiting for completion (`block=true`)

2. **Multiple Task Types**: Handles output retrieval for three distinct task types:
   - `local_bash` — shell command tasks
   - `local_agent` — local agent tasks (prefers in-memory `result.content`)
   - `remote_agent` — remote session tasks

3. **Backwards Compatibility**: The tool responds to legacy names (`AgentOutputTool`, `BashOutputTool`) that were renamed but may still be referenced

4. **Polling Mechanism**: When blocking, polls app state every 100ms with configurable timeout (max 600s)

5. **Structured Output**: Returns XML-tagged results with `<retrieval_status>`, `<task_id>`, `<output>`, `<errors>` for consistent parsing by callers

6. **UI Rendering**: Uses Ink (CLI React) components for rendering results in terminal environments with support for verbose/standard display modes

7. **Deprecated**: Tool documentation notes users should "prefer Read tool on the task output file path" for future use
