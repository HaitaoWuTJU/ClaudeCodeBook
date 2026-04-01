# Summary of `hooks/toolPermission/`

## Purpose of `toolPermission/`

This directory implements the **permission resolution system** for tool use in the agent/CLI application. It orchestrates the complete lifecycle of permission requests—from initial hooks and classifier checks through user interaction to final logging and persistence—ensuring tools execute only when authorized.

## Contents Overview

| File | Responsibility |
|------|----------------|
| `PermissionContext.js` | Core infrastructure: creates frozen context objects with methods for logging, persistence, hook execution, classifier auto-approval, queue management, and decision construction |
| `permissionLogging.ts` | Centralized analytics: fans out permission decisions to Statsig events, OpenTelemetry telemetry, code-edit metrics counters, and in-memory decision storage |
| `handlers/coordinatorHandler.ts` | Coordinator mode: automated-first resolution (hooks → classifier → fall through to dialog) |
| `handlers/interactiveHandler.ts` | Main-agent mode: interactive race (dialog + bridge/channels/hooks/classifier, await user) |
| `handlers/swarmWorkerHandler.ts` | Swarm worker mode: delegates decisions to swarm leader via mailbox |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│              External Caller (resolveToolPermission)             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  createPermissionContext() [PermissionContext.js]                │
│  • Builds PermissionContext with all resolution methods          │
│  • Bridges React queue state to generic PermissionQueueOps       │
│  • Provides: logDecision, persistPermissions, runHooks,         │
│    tryClassifier, handleUserAllow, cancelAndAbort, etc.          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Handler Selection (based on agent mode + permissionMode)         │
│                                                                 │
│  ┌───────────────────────┐                                       │
│  │ coordinatorHandler   │ permissionMode='local'                │
│  │   → hooks → classifier│ → returns decision or null           │
│  └───────────────────────┘                                       │
│  ┌───────────────────────┐                                       │
│  │ interactiveHandler    │ permissionMode='ask'                  │
│  │   → push dialog       │ → races all sources, awaits user      │
│  │     → user responds   │                                       │
│  └───────────────────────┘                                       │
│  ┌───────────────────────┐                                       │
│  │ swarmWorkerHandler    │ agent is swarm worker                  │
│  │   → classifier        │ → forwards to leader via mailbox      │
│  └───────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  logPermissionDecision() [permissionLogging.ts]                   │
│  • Extracts source, calculates wait time                          │
│  • FANS OUT to:                                                   │
│    1. Analytics (logEvent) — distinct event names per source      │
│    2. OTel telemetry (logOTelEvent)                               │
│    3. Code-edit metrics counter (code editing tools only)         │
│    4. toolUseContext.toolDecisions Map (in-memory persistence)   │
└─────────────────────────────────────────────────────────────────┘
```

**Data Flow Through the System:**

1. **Context Creation**: `createPermissionContext()` instantiates a frozen `PermissionContext` with all orchestration methods, queue bridge, and typed decision builders.

2. **Handler Resolution**: Based on agent mode and `permissionMode`, one of three handlers is invoked, each racing different approval sources using `createResolveOnce` to prevent race conditions.

3. **Approval Sources** (in priority/race order):
   - **Config hooks** (`runHooks`) — fast, synchronous, user-defined rules
   - **Classifier** (`tryClassifier`) — AI-powered auto-approval (Bash tool only, feature-flagged)
   - **Local user** (`handleUserAllow`) — human in the loop
   - **Bridge/CCR** — remote permission services
   - **Channels** — alternative permission sources

4. **Decision Handling**: When a source claims the permission:
   - `handleUserAllow` or `handleHookAllow` logs + persists to storage
   - `cancelAndAbort` handles rejection/cancellation
   - All paths call `logPermissionDecision()` for telemetry

5. **Logging Fan-Out**: Every decision triggers four logging destinations simultaneously (analytics, telemetry, metrics, in-memory map).

## Key Takeaways

1. **Multi-tier resolution**: The system prioritizes speed (hooks → ~50ms) over accuracy (classifier) over user experience (interactive), with graceful degradation when automated checks fail.

2. **Race safety**: `createResolveOnce` (with atomic `claim()`) ensures exactly one source wins when multiple permission sources respond simultaneously.

3. **Feature-gated intelligence**: Classifier auto-approval is behind `BASH_CLASSIFIER` and `TRANSCRIPT_CLASSIFIER` flags, allowing gradual rollout without affecting existing permission flows.

4. **Decoupled architecture**: `PermissionQueueOps` interface isolates React state from the core permission logic, enabling reuse in REPL/non-React environments.

5. **Comprehensive telemetry**: Every permission decision—approved or rejected, automated or manual—is logged to analytics, OTel, and code-edit metrics for observability and debugging.

6. **Hierarchical control**: Swarm workers delegate decisions to leaders, enabling centralized policy enforcement across distributed agents.

7. **Graceful fallthrough**: Handler chaining (coordinator → interactive, swarm timeout → interactive) ensures users can always approve a tool even when automated systems fail.
