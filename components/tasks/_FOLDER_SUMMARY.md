# Summary of `components/tasks/`

## Purpose of `components/tasks/`

The `tasks/` directory contains React/Ink components for displaying and managing various task types within a terminal-based CLI application. It provides:

1. **Dialog views** for detailed task inspection (shell commands, async agents, background tasks)
2. **Inline rendering** for task list items in the main interface
3. **Shared utilities** for consistent status iconography, colors, and activity descriptions
4. **Keyboard integration** for navigating, dismissing, and interacting with tasks

The directory acts as the presentation layer for the application's background job system, supporting multiple task types:

| Task Type | Subdirectory | Description |
|-----------|--------------|-------------|
| `local_bash` | `LocalShellTask/` | Local shell commands |
| `remote_agent` | `RemoteAgentTask/` | Remote agent sessions |
| `local_agent` | `LocalAgentTask/` | Local agent (with panel/async variants) |
| `in_process_teammate` | `InProcessTeammateTask/` | In-process teammate agents |
| `local_workflow` | `LocalWorkflow/` | Sequential workflow tasks |
| `monitor_mcp` | `MonitorMcp/` | MCP server monitors |
| `dream` | `Dream/` | Dream/session tasks |

## Contents Overview

### Dialog Components
| File | Purpose |
|------|---------|
| `BackgroundTasksDialog.tsx` | Master dialog for all background task management — list/detail view switcher, keyboard navigation, kill actions for all task types |
| `ShellDetailDialog.tsx` | Detailed view of a single shell task — shows status, runtime, command, and last 10 lines of output (polling every 1s for running tasks) |
| `AsyncAgentDetailDialog.tsx` | Detailed view of an async agent task — shows progress, recent activities, plan/prompt content, elapsed time, token/tool counts, errors |

### Inline Rendering
| File | Purpose |
|------|---------|
| `BackgroundTask.tsx` | Renders a single task item in a list/panel — handles all 8 task types with appropriate formatting, truncation, status icons |
| `ShellProgress.tsx` | Compact status badge for shell tasks — displays "done", "error", "stopped", or spinning indicator |

### Utilities
| File | Purpose |
|------|---------|
| `taskStatusUtils.tsx` | Shared helpers: `isTerminalStatus()`, `getTaskStatusIcon()`, `getTaskStatusColor()`, `describeTeammateActivity()`, `shouldHideTasksFooter()` |
| `renderToolActivity.tsx` | Renders tool activity items with icons and truncated descriptions |

### Root Component
| File | Purpose |
|------|---------|
| `Task.tsx` | Main tasks panel component — renders the header, keyboard shortcuts bar, and task list with footer controls |

## How Files Relate to Each Other

```
Task.tsx (root panel)
    │
    ├── BackgroundTask.tsx (list item rendering for each task type)
    │       └── ShellProgress.tsx (shell-specific status badge)
    │
    ├── BackgroundTasksDialog.tsx (all-tasks master dialog)
    │       ├── ShellDetailDialog.tsx (shell detail view)
    │       ├── AsyncAgentDetailDialog.tsx (async agent detail view)
    │       └── renderToolActivity.tsx (activity rendering)
    │
    └── taskStatusUtils.tsx (shared status utilities)
            └── getTaskStatusIcon(), getTaskStatusColor(), describeTeammateActivity()
```

**Data flow pattern:**
1. `Task.tsx` reads task state from `useAppState()`
2. `BackgroundTask.tsx` renders each task using `taskStatusUtils.tsx` for icons/colors
3. `BackgroundTasksDialog.tsx` is triggered by keyboard shortcut (↑↓ / Enter)
4. User navigates to `ShellDetailDialog` or `AsyncAgentDetailDialog` for specific task info
5. Detail dialogs poll file-based output or read agent state for real-time updates

## Key Takeaways

- **Ink-based TUI**: All components use the Ink (`ink.js`) React renderer for CLI display, not the DOM.
- **React Compiler**: All React components use `react/compiler-runtime` (`_c`, `$` cache array) for automatic memoization, indicating adoption of React's experimental compiler.
- **Feature-gated imports**: Optional task types (teammate, workflow, MCP, dream) are conditionally imported via `feature()` checks to avoid bundling unused code.
- **File-based output**: Shell task output is read from disk (`src/utils/task/diskOutput.js`) rather than stored in memory — only the last 8 KB is read for detail views.
- **Polling for live updates**: `ShellDetailDialog` and `AsyncAgentDetailDialog` poll their respective data sources every 1 second for running tasks using `useDeferredValue` to keep the UI responsive.
- **Keyboard-driven navigation**: All dialogs integrate with `useKeybindings` for consistent keyboard shortcuts (Space/Enter/Escape to close, Left to go back, X to kill, arrow keys to navigate).
- **Consistent status system**: `taskStatusUtils.tsx` provides a unified status abstraction with priority-based icon/color resolution (error > approval > shutdown > idle > status) used by all rendering components.
- **Two-level detail pattern**: The dialog system uses a list/detail split — `BackgroundTasksDialog` manages the list, while task-specific detail dialogs are opened for in-depth inspection.
