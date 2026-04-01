# Summary of `tasks/DreamTask/`

## Purpose of `DreamTask/`

Provides the UI-facing "wrapper" for an invisible background memory consolidation process. The `DreamTask.ts` file acts as a bridge between the auto-dream agent's silent work and the user's awareness—making long-running consolidation visible in the footer and Shift+Down panel without modifying the dream agent itself.

## Contents Overview

### `DreamTask.ts` (single file, ~5 KB)

Defines a **Task entry** for the task registry with full lifecycle management:

| Section | Purpose |
|---------|---------|
| **Type definitions** | `DreamTurn`, `DreamPhase`, `DreamTaskState` — mirrors the 4-stage dream phase but only tracks `'starting'` and `'updating'` for UI |
| **State management** | Functions to register, update, complete, and fail dream tasks via `updateTaskState` |
| **Turn tracking** | `addDreamTurn()` — accumulates assistant turns with text + collapsed tool counts, deduplicates new file paths, enforces 30-turn cap |
| **Phase logic** | Automatically advances phase from `'starting'` → `'updating'` when first Edit/Write tool is seen |
| **Kill handler** | Rewinds the consolidation lock so the next session can retry consolidation |

## How Files Relate to Each Other

```
DreamTask.ts
├── imports consolidationLock.js
│   └── rollbackConsolidationLock() — used in kill() to reset lock mtime
├── imports Task.js
│   └── Provides Task interface, TaskStateBase, createTaskStateBase, generateTaskId, SetAppState
└── imports framework.js
    └── Provides registerTask() and updateTaskState() — core registry/read/write primitives
```

The file has **no internal subdirectory files**—it is a self-contained unit that plugs into the task registry and delegates to shared framework utilities.

## Key Takeaways

1. **No parsing of dream prompts** — The task only reacts to tool use patterns, not LLM output content. It does not infer which of the 4 consolidation stages is running.
2. **Optimistic file tracking** — `filesTouched` is intentionally incomplete (misses writes via bash), serving as a rough progress indicator rather than an accurate manifest.
3. **Lock rollback is the primary kill action** — Instead of aborting a running process (the consolidation runs in a separate session), `kill()` rewinds the mtime so the next session's lock check passes and triggers retry.
4. **`notified: true` on completion** — Since the auto-dream has no model-facing notification mechanism, it immediately marks tasks as notified, skipping any notification queue logic.
5. **Turn cap prevents unbounded memory growth** — Keeping only 30 turns ensures the UI remains responsive even during very long consolidation runs.
