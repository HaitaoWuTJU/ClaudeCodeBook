# Summary of `coordinator/`

## Summary of `coordinator/`

### Purpose of `coordinator/`

The `coordinator/` directory houses the coordinator mode implementation for Claude Code's multi-agent orchestration system. This system enables a primary "coordinator" agent to direct multiple worker agents through research, implementation, and verification of code changes. The coordinator acts as a project manager that delegates tasks, synthesizes results, and ensures quality through iterative verification cycles.

### Contents Overview

The directory contains a single core file:

- **`coordinatorMode.ts`** — Implements all coordinator mode functionality including:
  - Feature flag detection and validation
  - Session mode synchronization for resumed sessions
  - Context building for worker agents
  - System prompt generation for the coordinator role

### How Files Relate to Each Other

Since `coordinator/` contains only one file, the relationships are external:

```
coordinatorMode.ts
├── Reads from: Environment variables (CLAUDE_CODE_COORDINATOR_MODE, CLAUDE_CODE_SIMPLE)
├── Queries: Statsig feature gates (tengu_scratch)
├── Consumes: Tool constants from ../tools/*/constants.js and ../constants/tools.js
├── Integrates with: Analytics service (logEvent)
├── Outputs to: Environment variables (for session resumption)
└── Consumed by: QueryEngine.ts (for context building and system prompt)
```

The file uses dependency injection for `scratchpadDir` (passed from `QueryEngine.ts`) to avoid circular import issues with filesystem modules.

### Key Takeaways

1. **Feature-gated architecture**: Coordinator mode requires both a Statsig feature flag (`tengu_scratch`) AND an environment variable to activate, providing two layers of control.

2. **Session resumption support**: The `matchSessionMode()` function synchronizes coordinator mode state when resuming sessions by writing directly to the environment variable.

3. **Worker capability modes**: Worker agents operate in two capability modes — "simple" (limited to Bash/Read/Edit) or "full" (all standard tools, MCP tools, and skills) — controlled by the `CLAUDE_CODE_SIMPLE` environment variable.

4. **Internal tool protection**: The `INTERNAL_WORKER_TOOLS` set explicitly reserves `TeamCreate`, `TeamDelete`, `SendMessage`, and `SyntheticOutput` for coordinator-only use, preventing workers from spawning additional agents.

5. **Concurrency-first design**: The system prompt emphasizes parallel task execution as a key principle, with the coordinator delegating independent subtasks simultaneously.

6. **Circular dependency mitigation**: The file duplicates `isScratchpadGateEnabled()` logic to avoid importing from `utils/permissions/filesystem.ts`, which would create a circular dependency chain.

7. **Analytics integration**: Session mode switches are tracked via `logEvent('tengu_coordinator_mode_switched')` for growth experimentation measurement.
