# Summary of `utils/task/`

## Purpose of `utils/task/`

This directory provides the **task output pipeline** for a Claude Code application — a multi-stage system that captures shell command and hook output, manages it with bounded memory via disk overflow, exposes incremental progress via polling, and surfaces results to the CLI and SDK with configurable truncation.

## Contents Overview

| File | Responsibility |
|------|----------------|
| **`diskOutput.ts`** | Async write queue with a **5 GB disk cap**; provides per-task `DiskTaskOutput` instances that write chunks to `{projectTemp}/{sessionId}/tasks/{taskId}.output` |
| **`TaskOutput.ts`** | Single source of truth for **shell command output**; routes data to file or buffers; manages overflow to `DiskOutput`; exposes `getStdout()` and a shared **polling tick** for CLI progress updates |
| **`framework.ts`** | **Polling loop** (`pollTasks`) that reads running tasks' disk deltas, evicts terminal tasks with grace periods, updates `AppState`, and enqueues `notification` messages |
| **`outputFormatting.ts`** | **Truncation layer** — enforces `TASK_MAX_OUTPUT_LENGTH` (default 32 kB, cap 160 kB) on output returned to the API; preserves tail and injects path header |
| **`sdkProgress.ts`** | Thin adapter that formats **SDK `task_progress` events** and enqueues them; shared by background agents and workflow batches |

## How Files Relate to Each Other

```
Shell Command / Hook
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│                     TaskOutput.ts                         │
│  ├─ stdoutToFile=true (bash) → write directly to fd       │
│  │      └─ #tick() polls tail every 1 s → enqueue         │
│  │         notification / update AppState                 │
│  └─ stdoutToFile=false (hooks) → #writeBuffered()         │
│         ├─ Within maxMemory (8 MB) → CircularBuffer       │
│         └─ Exceeds maxMemory → spillToDisk()              │
│            └─► DiskTaskOutput.append()                    │
└──────────────────────┬────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────┐
│                    diskOutput.ts                          │
│  ├─ Flat write queue with drain loop                      │
│  ├─ O_NOFOLLOW on open (symlink attack guard)             │
│  └─ 5 GB per-task cap (truncates with notice)            │
└──────────────────────┬────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────┐
│                   framework.ts                            │
│  pollTasks() — reads disk deltas via getTaskOutputDelta() │
│  ├─ Applies offset patches to AppState.tasks              │
│  ├─ Evicts terminal tasks (grace periods: 3 s / 30 s)    │
│  └─ Enqueues notification messages                         │
└──────────────────────┬────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        ▼                              ▼
┌───────────────────┐      ┌───────────────────────────────┐
│  sdkProgress.ts   │      │   outputFormatting.ts         │
│  emitTaskProgress()│      │   formatTaskOutput()          │
│  └─► SDK events   │      │   ├─ Truncate > 32 kB          │
│                   │      │   ├─ Preserve tail             │
│                   │      │   └─ Inject path header        │
└───────────────────┘      └───────────────────────────────┘
```

**Session stability**: `diskOutput.ts` captures the session ID on first call and never refreshes it. This prevents background tasks that survive `/clear` (which calls `regenerateSessionId()`) from orphaning their output files.

## Key Takeaways

1. **Memory-bounded pipeline**: Output flows either through an in-memory `CircularBuffer` (8 MB) or directly to a disk file, with automatic spillover — the process never accumulates unbounded heap from a long-running command.
2. **5 GB per-task disk cap**: `diskOutput.ts` guards against disk-fill attacks by appending a truncation notice once the cap is hit and dropping further chunks.
3. **Symlink-safe writes**: `O_NOFOLLOW` on every `open()` prevents sandbox attackers from redirecting writes via malicious symlinks in the tasks directory.
4. **Polling-based visibility**: Rather than event-driven callbacks, the CLI uses a shared `setInterval` (1 s, `.unref()`) that tail-reads output files. This avoids coupling between the shell subsystem and the framework layer.
5. **Avoids double delivery**: `framework.ts` deliberately **skips** completed tasks in `generateTaskAttachments` — each task type (`BackgroundTask`, `LocalSubagent`, etc.) is responsible for emitting its own completion notification to prevent duplicate messages.
6. **Grace periods for UI**: Terminal tasks (killed/failed/completed) are displayed for 3 s before eviction; coordinator panels (local_agent) are retained for 30 s.
7. **SDK and CLI share the same source**: `getStdout()` / `getTaskOutputDelta()` are used by both `formatTaskOutput()` (CLI API) and `emitTaskProgress()` (SDK events), ensuring consistent output representation.
8. **ENOENT resilience**: Read operations treat a missing output file as a valid (empty) state rather than an error — prevents reminder confusion when a tool result is small or when cross-session cleanup races with polling.
