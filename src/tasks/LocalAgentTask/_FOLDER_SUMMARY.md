# Summary of `tasks/LocalAgentTask/`

## Purpose of `LocalAgentTask/`

This directory houses the implementation for **background sub-agent lifecycle management** in the application's task system. Its sole concern is the `local_agent` task type: tracking progress, routing messages between the agent's API input and the UI transcript, handling completion notifications, and providing kill/cleanup controls. It is the backbone for features like teammate views, panel progress pills, and the "kill all agents" command.

## Contents Overview

| File | Purpose |
|------|---------|
| `LocalAgentTask.tsx` | Implements all task logic, state shapes, and helpers for `local_agent` tasks |

The directory contains exactly one file — `LocalAgentTask.tsx` (~83 KB) — which is the canonical implementation of the local agent task contract.

## How Files Relate to Each Other

Since the directory contains only one file, all relationships are **internal** to `LocalAgentTask.tsx`. The module forms a layered stack:

```
Exports (Task object, type guards, kill helpers)
        │
        ▼
Message & Notification Layer
  queuePendingMessage → drainPendingMessages → appendMessageToLocalAgent
  enqueueAgentNotification (XML serialization, guarded one-shot)
        │
        ▼
Progress Tracking Layer
  ProgressTracker (mutable) → getProgressUpdate → AgentProgress (immutable snapshot)
  updateProgressFromMessage (token accounting, tool activity extraction)
        │
        ▼
State Management Layer
  isLocalAgentTask / isPanelAgentTask (predicates)
  updateAgentProgress / updateAgentSummary (state mutations)
  killAsyncAgent / killAllRunningAgentTasks (lifecycle termination)
        │
        ▼
Task Export
  LocalAgentTask: Task — wires everything into the global task registry
```

## Key Takeaways

1. **Two message channels**: `pendingMessages` (strings queued for the next agent API call) and `messages` (the display transcript updated via `appendMessageToLocalAgent`). They serve different consumers and must not be conflated.

2. **Token and tool accounting**: `ProgressTracker` accumulates input/output token counts and maintains a sliding window of up to 5 recent tool activities with pre-resolved human-readable descriptions.

3. **One-shot notification guard**: `notified` is set atomically before enqueueing to prevent duplicate `<task-notification>` blocks from concurrent completion paths.

4. **Speculation abort**: Notifying a panel agent aborts any active prompt speculation because the pre-computed results may reference now-invalid task output.

5. **`SyntheticOutputTool` is invisible**: Tool calls of this name are excluded from `recentActivities` so they do not clutter the progress preview shown to users.

6. **`retain` blocks eviction**: Explicit retain (via `enterTeammateView`) clears the eviction deadline and enables disk bootstrap; the eviction timer only applies when the UI is closed or the task is passively abandoned.

7. **All panel/pill filters must agree on `isPanelAgentTask`**: The gate (excludes `main-session`) is centralized here so that any future change is a single-point modification.

8. **Background summarization is non-destructive**: `updateAgentProgress` deliberately preserves the existing `summary` field; `updateAgentSummary` is the only function allowed to overwrite it.
