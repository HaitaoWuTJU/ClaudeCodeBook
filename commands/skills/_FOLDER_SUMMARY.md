# Summary of `commands/skills/`

## Purpose of `skills/`

Implements the "Skills" command, which displays an interactive menu of available skills to the user. The command uses a lazy-loaded, component-based architecture.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Command configuration — declares the command name, type, and lazy-loaded import path |
| `skills.tsx` | Command handler — renders the `SkillsMenu` component with context data |

## How Files Relate to Each Other

```
┌─────────────────────────────┐
│  index.ts                   │
│  - Defines command metadata │
│  - load: () => import()     │──────────────┐
└─────────────────────────────┘              │
                                               │ lazy import
                                               ▼
┌─────────────────────────────┐
│  skills.tsx                 │
│  - call() async function    │
│  - Renders <SkillsMenu />   │──────────────┐
└─────────────────────────────┘              │
                                               │ import
                                               ▼
                                      ┌──────────────┐
                                      │ SkillsMenu   │
                                      │ (component)  │
                                      └──────────────┘
```

1. When the skills command is invoked, `index.ts` triggers a dynamic import of `skills.tsx`
2. `skills.tsx` receives the command context (available commands) and an `onDone` callback
3. It renders `SkillsMenu`, passing commands as props and `onDone` as `onExit`

## Key Takeaways

- **Lazy loading**: The command implementation is split into a separate file loaded only when needed, keeping the initial bundle small.
- **Command pattern**: Consistent with the repository's architecture — each command has a config file and an entry point function (`call`).
- **Context-driven**: The handler receives commands from the command context, enabling dynamic menu population.
- **Callback-based exit**: The `onDone` callback is wired to `onExit`, allowing the SkillsMenu to notify the command system when the user exits.
