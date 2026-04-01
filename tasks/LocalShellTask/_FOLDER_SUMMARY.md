# Summary of `tasks/LocalShellTask/`

## Purpose of `LocalShellTask/`

Manages the full lifecycle of local bash/shell commands executed by the CLI agent—spawning, backgrounding, stall detection for interactive prompts, completion notifications, and cleanup.

## Contents Overview

| File | Role |
|------|------|
| `LocalShellTask.tsx` | Core implementation: spawns shell tasks, starts stall watchdogs, enqueues completion notifications, and exposes task lifecycle functions (`spawnShellTask`, `backgroundTask`, `backgroundAll`, etc.) |
| `killShellTasks.ts` | Pure (non-React) kill helpers that terminate shell processes by ID or by agent ownership; decoupled from React/Ink so `runAgent.ts` can use it |
| `guards.ts` | Shared type definitions and `isLocalShellTask` type guard for non-React consumers that need to work with `LocalShellTaskState` objects |

## How Files Relate to Each Other

```
LocalShellTask.tsx
    │
    ├── imports ──► guards.ts
    │               └── isLocalShellTask (type guard)
    │               └── LocalShellTaskState (type)
    │               └── BashTaskKind (type)
    │
    ├── imports ──► killShellTasks.ts
    │               └── killTask (synchronous termination)
    │
    └── exports ──► killShellTasks.ts
                        │
                        └── imported by runAgent.ts finally block
                            (agent-scoped cleanup)
```

**Decoupling rationale:** `runAgent.ts` cannot depend on React/Ink, but needs to kill shell tasks when agents exit. By extracting `isLocalShellTask` into `guards.ts` and kill logic into `killShellTasks.ts`, the module graph stays clean.

## Key Takeaways

1. **Stall detection for interactive prompts** — The 45-second watchdog differentiates slow-but-healthy commands from blocked interactive prompts (`(y/n)`, `Press any key`, etc.), sending a non-terminal notification to guide the user.

2. **Dual task ownership** — Tasks track both `agentId` (who spawned them) and `shellCommand.kill` (process control). If an agent dies unexpectedly, orphaned tasks are cleaned up by `runAgent.ts` via `killShellTasksForAgent`.

3. **Ctrl+B context sensitivity** — If any bash or agent task is foregrounded, Ctrl+B backgrounds the task(s); otherwise it backgrounds the entire session.

4. **Notification deduplication** — The `notified` flag on task state prevents double-sends when `TaskStopTool` races with the shell command's result promise.

5. **Feature-gated monitor streams** — `MONITOR_TOOL` controls whether `spawnShellTask` reports as type `'monitor'` or `'bash'`, with distinct summary strings and notification priorities.
