# Summary of `commands/tasks/`

## Purpose of `tasks/`

Provides a **background tasks management command** that allows users to list and manage background tasks running in the IDE. The directory implements a lazy-loaded command with a dialog-based UI for task management.

## Contents Overview

| File | Role | Purpose |
|------|------|---------|
| `tasks.ts` | Command definition | Declares the command metadata (name, aliases, description, type) and lazy-loads the implementation |
| `tasks.tsx` | Command implementation | Entry point that renders the `BackgroundTasksDialog` component with necessary context |

## How Files Relate to Each Other

```
tasks.ts (Command Definition)
│
├── Defines command metadata:
│   ├── name: 'tasks'
│   ├── aliases: ['bashes']
│   └── type: 'local-jsx'
│
├── Exports default: tasks (satisfies Command)
│
└── Uses lazy loading ─────────────────┐
                                        ▼
                               tasks.tsx (Command Implementation)
                               │
                               ├── Receives: onDone callback, context
                               │
                               └── Returns: <BackgroundTasksDialog />
```

**Relationship**: `tasks.ts` acts as the command registry entry point, while `tasks.tsx` provides the actual implementation. The lazy `load()` function defers importing `tasks.tsx` until the command is first invoked, improving startup performance.

## Key Takeaways

1. **Lazy Loading Pattern**: The command implementation is dynamically imported only when needed, keeping the initial bundle lean.

2. **Command Interface**: Both files follow the `LocalJSXCommand` pattern with a `call(onDone, context)` function signature for VS Code-style extension integration.

3. **User-Facing Features**:
   - Command accessible via `tasks` or `bashes` aliases
   - Renders a dialog (`BackgroundTasksDialog`) for managing background jobs

4. **Separation of Concerns**:
   - **Metadata** (naming, aliases, description) lives in `tasks.ts`
   - **Behavior** (rendering, interaction) lives in `tasks.tsx`
