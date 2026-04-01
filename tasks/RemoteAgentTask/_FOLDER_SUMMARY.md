# Summary of `tasks/RemoteAgentTask/`

# RemoteAgentTask Directory Analysis

## Purpose of `RemoteAgentTask/`

This directory implements a **remote agent task orchestration system** that manages long-running tasks executed in cloud environments via Teleport. The system handles the complete lifecycle of remote tasks:

- Spawning tasks in cloud environments
- Polling for session events and progress
- Parsing structured results (plans, reviews) from session logs
- Notifying the local model when tasks complete, fail, or require input

The system supports multiple task types: standard remote agents, Ultraplan (AI planning), UltraReview (automated code review), and background PR operations.

---

## Contents Overview

### File Structure

```
RemoteAgentTask/
├── RemoteAgentTask.ts       # Main task implementation (126K+ chars)
├── RemoteAgentTaskState.ts  # State type definitions
├── registerCompletionChecker.ts
└── types.ts
```

### Key Files

| File | Purpose |
|------|---------|
| `RemoteAgentTask.ts` | Core task class extending `Task`, handles session polling, log extraction, notification dispatch |
| `RemoteAgentTaskState.ts` | State type extending `TaskStateBase` with `sessionId`, `remoteTaskType`, `todoList`, `log[]`, `reviewProgress`, `ultraplanPhase` |
| `registerCompletionChecker.ts` | Completion checker registry per task type |
| `types.ts` | Shared type definitions including `RemoteTaskType`, `RemoteTaskMetadata`, `RemoteAgentPreconditionResult` |

---

## How Files Relate to Each Other

```
RemoteAgentTaskState.ts / types.ts
         │
         ▼ (provides type definitions)
RemoteAgentTask.ts
         │
         ├──► registerCompletionChecker.ts (completion checkers registry)
         │
         ├──► Task.ts (base class)
         │
         ├──► utils/teleport.js (session management)
         │         │
         │         └──► pollRemoteSessionEvents()
         │         └──► fetchSession()
         │         └──► archiveRemoteSession()
         │
         ├──► utils/task/diskOutput.js (output persistence)
         │         │
         │         └──► appendTaskOutput()
         │         └──► initTaskOutput()
         │
         ├──► utils/sessionStorage.js (metadata survival)
         │         │
         │         └──► writeRemoteAgentMetadata()
         │         └──► listRemoteAgentMetadata()
         │
         └──► utils/messageQueueManager.js (notifications)
                   │
                   └──► enqueuePendingNotification()
```

### Type Flow

1. **`types.ts`** defines `RemoteTaskType` (enum union) and `RemoteAgentPreconditionResult`
2. **`RemoteAgentTaskState.ts`** imports those types and extends `TaskStateBase`
3. **`RemoteAgentTask.ts`** uses the state types and implements the `Task` interface

---

## Key Takeaways

### 1. Multi-Task Architecture
The system handles **5 distinct task types** through a unified polling mechanism:

| Task Type | Purpose | Completion Trigger |
|-----------|---------|-------------------|
| `remote-agent` | General-purpose remote execution | Custom checkers |
| `ultraplan` | AI planning sessions | `<ultraplan>` XML tag extraction |
| `ultrareview` | Automated code review | `<remote-review>` tag + review progress |
| `autofix-pr` | PR automation | Task-specific logic |
| `background-pr` | Background PR operations | Eligibility check |

### 2. Session Persistence for `--resume`
Remote agent metadata is written to **sessionStorage** via sidecar files, enabling task resumption after interruption:

```typescript
// Spawn: persist for resume
persistRemoteAgentMetadata(taskId, { sessionId, taskType, ... })

// Resume: read from sidecar
listRemoteAgentMetadata().find(m => m.taskId === taskId)
```

### 3. Three-Tier Review Extraction
The review parser has **progressive fallback levels**:

1. **Hook progress scan** (bughunter path) → `extractReviewFromLog`
2. **Assistant message scan** (prompt mode) → `extractReviewFromLog`
3. **Full concatenation fallback** → `extractReviewTagFromLog` (no fallback, prevents premature completion)

### 4. Atomic Notification Deduplication
The `markTaskNotified()` function uses **atomic check-and-set** to prevent duplicate notifications:

```typescript
// Prevents race condition: multiple poll ticks fire simultaneously
updateTaskState(taskId, (state) => {
  if (state.notified) return state; // already notified
  return { ...state, notified: true };
});
```

### 5. Precondition Checking
Before spawning any remote task, `checkRemoteAgentEligibility()` validates:

- User is logged in
- In a git repository
- GitHub app is installed (for review/PR tasks)
- Organization policy permits the operation

### 6. Event-Driven Architecture
The polling loop (`pollRemoteSessionEvents`) accumulates `SDKMessage[]` into the task log on each tick, then:

- Invokes registered completion checkers
- Extracts structured data from logs
- Triggers notifications when conditions are met

### 7. Output File Tags
Structured data is embedded in logs using **XML tags**:

| Tag | Purpose |
|-----|---------|
| `<ultraplan>...</ultraplan>` | Extracted plan content |
| `<remote-review>...</remote-review>` | Extracted review content |
| `<review-progress>...</review-progress>` | Review progress status |
| `<tool_output_file>...</tool_output_file>` | Output file paths |
