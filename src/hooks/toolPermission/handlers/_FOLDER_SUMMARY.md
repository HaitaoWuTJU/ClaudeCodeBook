# Summary of `hooks/toolPermission/handlers/`

## Summary of `handlers/`

### Purpose of `handlers/`

This directory contains the **permission resolution strategies** for tool execution in the agent system. Each handler implements a different **resolution pathway** that determines whether a tool (like bash commands) gets approved or rejected. The handlers are selected by the `resolveToolPermission` orchestrator based on the current agent mode (coordinator, main-agent, or swarm worker) and the `permissionMode` setting.

### Contents Overview

| File | Agent Mode | Resolution Strategy |
|------|------------|---------------------|
| `coordinatorHandler.ts` | Coordinator worker | **Automated-first**: Try hooks → try classifier → fall through to dialog |
| `interactiveHandler.ts` | Main agent | **Interactive race**: Show dialog, race bridge/channels/hooks/classifier, await user |
| `swarmWorkerHandler.ts` | Swarm worker | **Delegated to leader**: Try classifier → forward to swarm leader via mailbox |

### How Files Relate to Each Other

```
resolveToolPermission() [orchestrator]
       │
       ├── permissionMode === 'local'
       │       │
       │       └── coordinatorHandler.ts
       │              │
       │              ├── hooks succeed? → return PermissionDecision ✓
       │              ├── classifier succeeds? → return PermissionDecision ✓
       │              └── all fail → return null → triggers interactiveHandler.ts
       │
       ├── permissionMode === 'ask'
       │       │
       │       └── interactiveHandler.ts
       │              │
       │              ├── Push confirmation dialog
       │              ├── Race: bridge ←→ channels ←→ hooks ←→ classifier
       │              │         ↑
       │              │    (first to claim wins)
       │              └── User responds → return PermissionDecision
       │
       └── agent is swarm worker
               │
               └── swarmWorkerHandler.ts
                      │
                      ├── classifier succeeds? → return PermissionDecision ✓
                      ├── Forward to leader via mailbox
                      └── Await leader response (allow/reject/abort)
```

**Shared Patterns:**
- All use `createResolveOnce` to prevent race conditions from multiple resolution sources
- All accept similar parameters: `PermissionContext`, `description`, `updatedInput`, `suggestions`
- All return `PermissionDecision` on success, `null` to signal fallback to another handler

**Cross-cutting Concerns:**
- **Feature flags**: `BASH_CLASSIFIER`, `KAIROS`, `KAIROS_CHANNELS`, `TRANSCRIPT_CLASSIFIER` gate classifier/channel features
- **Error resilience**: Failures in automated checks → fall through to user interaction
- **Abort handling**: All handlers register abort listeners to prevent hanging

### Key Takeaways

1. **Tiered resolution**: The system prioritizes speed (hooks → 50ms) over accuracy (classifier → slower) over user experience (interactive → slowest)

2. **Flexibility**: The same permission request can be resolved by different handlers depending on context (local vs ask mode, worker vs leader agent)

3. **Race safety**: `createResolveOnce` ensures only one permission source wins, regardless of timing

4. **Delegation**: Swarm workers delegate decisions to leaders rather than making them locally, enabling hierarchical control

5. **Fallback chain**: `null` return values create graceful degradation paths (coordinator → interactive, swarm → leader timeout → interactive)
