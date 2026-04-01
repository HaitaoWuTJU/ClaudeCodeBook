# Summary of `tools/TaskListTool/`

## Purpose of `TaskListTool/`

This directory implements a task list tool for an agent/AI system. The tool allows AI agents to retrieve and display all non-internal tasks from a task list, providing a standardized interface for listing tasks within a larger agent or workflow system.

## Contents Overview

| File | Purpose |
|------|---------|
| `constants.ts` | Defines the canonical tool name identifier (`TaskList`) |
| `prompt.ts` | Generates dynamic prompt instructions for AI agents on when and how to use the tool |
| `TaskListTool.ts` | Core implementation that fetches, filters, and formats tasks for output |

## How Files Relate to Each Other

```
constants.ts (tool name identifier)
       ↓
TaskListTool.ts imports:
       ├── TASK_LIST_TOOL_NAME ──────────────► Used as tool identifier
       ├── DESCRIPTION, getPrompt() ◄───────── Imported from prompt.ts
       ├── listTasks() ◄────────────────────── Fetches raw task data
       └── buildTool() ◄────────────────────── Factory that assembles the tool
```

**Data flow**: `prompt.ts` provides the instruction text → `TaskListTool.ts` imports and uses it as the tool's `description` and `promptTemplate` → `constants.ts` provides the name that ties everything together.

## Key Takeaways

- **Conditional prompts**: The tool adapts its instructions based on whether agent swarms mode is enabled, showing different workflow guidance for teammates vs. solo agents
- **Smart filtering**: Tasks marked with `metadata._internal` are automatically hidden; completed tasks are filtered from blocker lists
- **Agent-first design**: The tool is designed for AI agent consumption—uses deferred execution, read-only access, and clear ordering preferences (lowest ID first)
- **Integration point**: This tool references other task tools (`TaskGet`, `TaskUpdate`) to enable sequential agent workflows
- **Strict schemas**: Both input (empty) and output schemas are strictly defined using Zod for validation
