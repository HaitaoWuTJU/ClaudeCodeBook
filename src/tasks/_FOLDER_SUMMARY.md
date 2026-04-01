# Summary of `tasks/`

## Purpose of `tasks/`

The `tasks/` directory is the **central registry and lifecycle layer** for all background work in the application. It provides a unified abstraction for disparate execution environments — local shells, local LLM agents, remote cloud sessions, in-process teammates, and automatic memory consolidation — so the rest of the codebase can interact with all background work through a consistent interface.

## Contents Overview

### Entry Point & Framework

| File | Purpose |
|------|---------|
| `index.ts` | Re-exports the public API (`getAllTasks`, `getBackgroundTasks`, `getTaskById`) from `Task.js`; imports from sibling subdirectories to avoid circular dependency issues |
| `Task.js` | Core task registry: `getTaskByType`, `registerTask`, `updateTaskState`; maintains `allTasks` map; handles task event forwarding |
| `types.js` | Union type `TaskState` (all concrete task types) and `BackgroundTaskState`; `isBackgroundTask()` predicate |
| `LocalMainSessionTask.ts` | Handles Ctrl+B backgrounding of the main session query |
| `stopTask.ts` | Shared stop logic for both `TaskStopTool` (LLM) and SDK `stop_task` |
| `pillLabel.ts` | Generates compact UI footer pill labels for background tasks |

### Task Implementations

| Directory | Task Type | Manages |
|-----------|-----------|---------|
| `LocalShellTask/` | `local_bash` / `monitor` | Spawned shell processes, stall detection, completion notifications |
| `LocalAgentTask/` | `local_agent` | Local sub-agent sessions, panel/pill display, message routing, kill/eviction |
| `InProcessTeammateTask/` | `in_process_teammate` | Teammate tasks inside leader process, per-turn abort control, team identity |
| `RemoteAgentTask/` | `remote_agent`, `ultraplan`, `ultrareview`, `autofix-pr`, `background-pr` | Cloud session polling, log extraction, precondition checking |
| `DreamTask/` | `dream` | Auto-dream consolidation visibility, turn accumulation, lock rollback on kill |

## How Files Relate to Each Other

```
                    ┌─────────────────────────────────────────────┐
                    │                 Task.js                      │
                    │  allTasks: Map<taskId, Task>                 │
                    │  registerTask(type, impl)                    │
                    │  updateTaskState(taskId, updater)            │
                    │         ▲           ▲                        │
                    └─────────┼───────────┼────────────────────────┘
                              │           │
          ┌───────────────────┼───────────┼────────────────────────┐
          │                   │           │                        │
   LocalShellTask/     LocalAgentTask/    RemoteAgentTask/        InProcessTeammateTask/
   killShellTasks.ts   LocalAgentTask.tsx RemoteAgentTask.ts      InProcessTeammateTask.tsx
          │                   │                    │                       │
          │              types.ts              RemoteAgentTaskState.ts   types.ts
          │                                                             │
          └──────────────────────► index.ts ◄───────────────────────────┘
                                      │
                                pillLabel.ts ←─── reads BackgroundTaskState[]
                                stopTask.ts  ←─── calls getTaskByType()
                         LocalMainSessionTask.ts
                                 DreamTask/
```

**Dependency flow:**
- `Task.js` imports all concrete task implementations to build the registry
- `index.ts` re-exports only the public API, hiding internal details from consumers
- Individual task files import from `Task.js`, `types.js`, and shared utilities
- `pillLabel.ts` and `stopTask.ts` read task state but do not mutate it directly

## Key Takeaways

### 1. Six-Tier Task Taxonomy

The system distinguishes **six task types** with distinct semantics:

| Type | Execution Location | UI Behavior | Kill Mechanism |
|------|---------------------|-------------|----------------|
| `local_bash` | Child process | Shell-style output | SIGTERM → SIGKILL |
| `monitor` | Child process | Passive output stream | SIGTERM → SIGKILL |
| `local_agent` | Local LLM API | Panel + pill | AbortController + SDK event |
| `in_process_teammate` | AsyncLocalStorage (in-process) | Team tree view | Dual abort (full turn vs. current work) |
| `remote_agent` | Cloud via Teleport | Shift+Down panel | Remote session archive |
| `dream` | Auto-dream agent | Footer pill | Lock mtime rollback |

### 2. Backgrounding Is Central to the UX

Three mechanisms support backgrounding:

- **Ctrl+B twice** (`LocalMainSessionTask.ts`): Backgrounds an active main session query, creating a new background task
- **"Go to background" button** (`LocalAgentTask.tsx`): Explicitly moves a panel agent to backgrounded state
- **`--background` flag** (`LocalShellTask.tsx`, `RemoteAgentTask.ts`): Spawns a task already in backgrounded state

Foreground/background swaps are atomic via `setAppState` updates that only touch the specific task's `status` and `isBackgrounded` fields.

### 3. Notification Deduplication Is Atomic

Every task type uses a **check-and-set** pattern on the `notified` flag before enqueueing notifications:

```typescript
updateTaskState(taskId, (state) => {
  if (state.notified) return state;  // atomic guard
  return { ...state, notified: true };
});
```

This prevents race conditions when `TaskStopTool`, SDK aborts, and normal completion fire simultaneously.

### 4. Memory Management Is Tiered

Tasks use multiple eviction strategies:

| Task Type | Eviction Trigger | Mechanism |
|-----------|------------------|-----------|
| `local_agent` (panel) | 5-minute idle after UI close | `setEvictionDeadline()` + `evictTaskOutput()` |
| `local_agent` (background) | 30-minute idle | `setEvictionDeadline()` |
| `in_process_teammate` | `TEAMMATE_MESSAGES_UI_CAP = 50` | `appendCappedMessage()` |
| `dream` | 30-turn cap | Hard limit in `addDreamTurn()` |
| `remote_agent` | No eviction | Lives in cloud; local state is metadata |

### 5. Task Identity Is Stable Across Restarts

Local agents and remote sessions persist metadata to sessionStorage sidecar files (`writeRemoteAgentMetadata`, `writeLocalAgentMetadata`). On restart, `listRemoteAgentMetadata()` / `listLocalAgentMetadata()` are consulted to rebuild task state, enabling the `--resume` flow.

### 6. SDK Events Are Emitted for All Task Terminations

Every kill or natural completion path calls `emitTaskTerminatedSdk()` with a consistent payload:

```typescript
emitTaskTerminatedSdk(taskId, reason, {
  exitCode,
  agentId,
  taskType,
  errorMessage,
});
```

This is the canonical event for SDK consumers monitoring task lifecycle.

### 7. Isolated Output for Background Tasks

Background tasks (especially `local_agent`) use **separate transcript files** (`getAgentTranscriptPath(taskId)`) rather than the main session transcript. This prevents corruption when `/clear` rewrites the session transcript while background work is still writing to it.
