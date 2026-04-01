# Summary of `commands/agents/`

## Purpose of `agents/`

The `agents/` directory implements a command module for managing and interacting with agent configurations. It provides a UI menu (`AgentsMenu`) that lists available tools based on the user's permission context, allowing them to select and execute agent-related operations.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command registration entry point with lazy-loading setup |
| `agents.tsx` | Main implementation: fetches tools and renders the `AgentsMenu` component |

## How Files Relate to Each Other

```
index.ts (command registration)
    │
    └── lazy-loads ──► agents.tsx (implementation)
                            │
                            ├── ToolUseContext (input)
                            ├── toolPermissionContext (from app state)
                            │
                            ├── getTools(context) ──► tool list
                            │
                            └── AgentsMenu component (output)
```

**Load Sequence:**
1. The command system loads `index.ts` and registers the command metadata
2. When the command is invoked, `load()` triggers a dynamic import of `agents.tsx`
3. `agents.tsx` executes the `call()` function which:
   - Retrieves the current app state and permission context
   - Fetches available tools filtered by permissions
   - Renders the `AgentsMenu` component

## Key Takeaways

- **Lazy Loading Pattern**: `index.ts` defers importing the full implementation until needed, optimizing bundle size and initial load time
- **Permission-Based Access Control**: Tools are dynamically fetched using `getTools(permissionContext)`, ensuring users only see tools they're authorized to use
- **Async Command Pattern**: The `call()` function is async and returns a `Promise<React.ReactNode>`, enabling dynamic initialization and potential async operations before rendering
- **Separation of Concerns**: Metadata/registration logic is isolated in `index.ts` while business logic lives in `agents.tsx`
- **Exit Handling**: The `onDone` callback is passed through as `onExit`, allowing the `AgentsMenu` to signal when the user has completed their interaction
