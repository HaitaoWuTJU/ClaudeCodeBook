# Summary of `tasks/InProcessTeammateTask/`

## Purpose of `InProcessTeammateTask/`

Provides the **state layer and lifecycle management** for in-process teammates — AI sub-agents that run inside the leader process (via AsyncLocalStorage), with team-aware identity, per-task abort control, plan-mode tracking, permission modes, and UI-safe message caps.

## Contents Overview

| File | Role |
|------|------|
| `types.ts` | Type definitions (`InProcessTeammateTaskState`, `TeammateIdentity`), constants (`TEAMMATE_MESSAGES_UI_CAP = 50`), type guard (`isInProcessTeammateTask`), and capped-array utility (`appendCappedMessage`) |
| `InProcessTeammateTask.tsx` | Public API functions: kill, shutdown request, message injection/append, and read-only queries (`findByAgentId`, `getAll`, `getRunningSorted`) |

## How Files Relate to Each Other

```
types.ts                           InProcessTeammateTask.tsx
┌──────────────────────────────┐   ┌─────────────────────────────────────────┐
│  InProcessTeammateTaskState  │   │  requestTeammateShutdown(taskId)        │
│  TeammateIdentity            │◄─│  → updates InProcessTeammateTaskState    │
│  isInProcessTeammateTask()   │   │    shutdownRequested field               │
│  TEAMMATE_MESSAGES_UI_CAP    │   │                                         │
│  appendCappedMessage<T>()    │   │  appendTeammateMessage(taskId, msg)     │
│         ▲                    │   │  → appendCappedMessage(prev, newMsg)     │
│         │                    │   │                                         │
│    imported by               │   │  injectUserMessageToTeammate(taskId,…)  │
└─────────┼────────────────────┘   │  → appends to pendingUserMessages queue  │
          └────────────────────────│                                         │
                                   │  findTeammateTaskByAgentId(agentId)     │
                                   │  getAllInProcessTeammateTasks()         │
                                   │  getRunningTeammatesSorted()            │
                                   │  → read from AppState; filter by status │
                                   └─────────────────┬───────────────────────┘
                                                     │
                                         uses updateTaskState() from
                                         ../../utils/task/framework.js
                                         uses killInProcessTeammate() from
                                         ../../utils/swarm/spawnInProcess.js
```

**Data flow summary**: `InProcessTeammateTask.tsx` reads and mutates `InProcessTeammateTaskState` records in AppState. It imports type definitions and the capped-append utility from `types.ts`; uses `updateTaskState()` for writes; uses `killInProcessTeammate()` for termination.

## Key Takeaways

1. **Two abort controllers per task**: `abortController` terminates the entire teammate; `currentWorkAbortController` aborts only the current turn — this distinction is essential for graceful turn-level cancellation without killing the teammate.

2. **Memory cap is deliberate and measured**: `TEAMMATE_MESSAGES_UI_CAP = 50` was set after BQ analysis showed message arrays dominate memory in long-running or high-agent-count scenarios (up to 36.8GB in burst loads). The UI zoomed transcript intentionally holds a capped view; the authoritative history lives in `inProcessRunner.allMessages` and on disk.

3. **Sort order must stay stable**: `getRunningTeammatesSorted()` sorts alphabetically by `agentName`. Multiple consumers (`TeammateSpinnerTree`, `PromptInput` footer, `useBackgroundTaskNavigation`) index into this array with `selectedIPAgentIndex`. Any change to the sort breaks these indices silently.

4. **Identity is stored in AppState**: `TeammateIdentity` (agentId, agentName, teamName, color, planModeRequired, parentSessionId) is persisted as part of the task state — not reconstructed on reload — ensuring continuity across leader restarts.

5. **Message injection is guarded by status**: Only `running` or `idle` tasks accept injected user messages; terminal statuses (`completed`, `killed`, `failed`) drop messages silently with debug logging to avoid crashes on stale references.
