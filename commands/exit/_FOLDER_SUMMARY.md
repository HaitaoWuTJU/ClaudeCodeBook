# Summary of `commands/exit/`

## Purpose of `exit/`

This directory implements the `/exit` and `/quit` commands for the Claude REPL, providing a clean user experience when leaving the interactive session. It handles multiple exit scenarios: normal termination, background session detachment, and worktree-specific flows.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command descriptor — defines metadata (`name`, `aliases`, `description`) and lazy-loads the implementation |
| `exit.tsx` | Command implementation — orchestrates the actual exit logic, session handling, and UI rendering |

## How Files Relate to Each Other

```
index.ts (descriptor)
    │
    ├── Defines: name='exit', aliases=['quit']
    ├── Sets: immediate=true, type='local-jsx'
    │
    └── load() → dynamic import('./exit.js')
                          │
                          └── exit.tsx (implementation)
                                  │
                                  ├── Detects background tmux session
                                  ├── Checks for active worktree session
                                  ├── Renders <ExitFlow> if in worktree
                                  └── Shows goodbye message + graceful shutdown
```

The **command descriptor** (`index.ts`) serves as the entry point for the command system. It provides metadata and defers actual execution to the **implementation** (`exit.tsx`) via dynamic `import()`. This lazy-loading pattern keeps the initial bundle smaller and allows the implementation to be loaded only when needed.

## Key Takeaways

- **Three exit paths**: Normal termination, tmux background detachment, and worktree-specific flow
- **Background sessions preserved**: When exiting from a detached tmux session, the REPL stays alive for reconnection via `claude attach`
- **Worktree integration**: Active worktree sessions trigger a special `ExitFlow` UI component instead of immediate exit
- **Lazy loading**: Implementation is dynamically imported to optimize startup performance
- **TypeScript with `satisfies`**: Uses modern TypeScript to validate command structure at compile time
