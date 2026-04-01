# Summary of `query/`

## Purpose of `query/`

Provides the runtime infrastructure for query execution, separating configuration, dependencies, hook orchestration, and budget management into distinct, composable modules. The directory supports the `query()` function by supplying immutable config, injectable dependencies, end-of-turn hook execution, and continuation decisions.

## Contents Overview

| File | Role |
|------|------|
| **`config.ts`** | Snapshots immutable runtime configuration (session ID, feature gates) once per `query()` call, separating it from mutable state |
| **`deps.ts`** | Dependency injection container exporting `QueryDeps` contract and `productionDeps()` factory for testability |
| **`stopHooks.ts`** | Async generator that executes Stop → TaskCompleted → TeammateIdle hooks at turn end, yielding stream events and processing hook messages/errors |
| **`tokenBudget.ts`** | Budget decision logic that evaluates token consumption against thresholds to produce continue/stop decisions |

## How Files Relate to Each Other

```
query.ts (orchestrator)
├── deps.ts ──────────────→ Provides callModel, microcompact, autocompact, uuid
├── config.ts ────────────→ Provides sessionId, fastModeEnabled, statsig gates
├── stopHooks.ts ←────────→ Called at turn end; runs hooks, yields stream events
└── tokenBudget.ts ←───────→ Called to decide whether to continue or halt execution
```

**Config flow:** `config.ts` reads session state and Statsig/env gates at query entry, producing an immutable snapshot passed into the query execution context. Comments in `config.ts` indicate this separation enables future extraction of `step()` as a pure `(state, event, config) → state` reducer.

**Deps flow:** `deps.ts` provides the dependency injection container. Tests inject fakes without per-module `spyOn` boilerplate; production uses `productionDeps()` returning real implementations (model calls, compaction, UUID generation).

**Hook flow:** `stopHooks.ts` is an async generator called at turn end. It conditionally loads feature-gated services (memory extraction, job classification, auto-dream), runs computer use cleanup for Chicago MCP, executes stop hooks, then teammate-specific hooks (TaskCompleted, TeammateIdle), yielding stream events/messages throughout.

**Budget flow:** `tokenBudget.ts` evaluates whether to continue by checking budget usage against `COMPLETION_THRESHOLD` (90%) and detecting diminishing returns (< 500 tokens delta for 3+ continuations), returning a `ContinueDecision` or `StopDecision`.

## Key Takeaways

- **Architectural intent:** Immutable config separated from mutable state; `step()` designed as a future pure reducer `(state, event, config) → state`
- **Subagent exclusion:** All hook execution and most background services skip when `toolUseContext.agentId` is set — subagents must not pollute the main thread's timeline or release process locks
- **Feature gates:** Env vars (`CLAUDE_CODE_*`) and Statsig (`tengu_streaming_tool_execution2`) gate heavy dependencies and optional features; `feature()` wrappers enable tree-shaking in external builds
- **Async generator pattern:** `stopHooks.ts` uses `async generator` to stream progress, attachments, and messages rather than returning a flat result
- **Dependency injection:** Explicit `QueryDeps` type with `typeof fn` signatures keeps test fakes in sync with real implementations automatically
- **Diminishing returns detection:** Budget system halts after 3+ continuations with delta tokens below 500, preventing wasted computation
