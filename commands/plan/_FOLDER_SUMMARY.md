# Summary of `commands/plan/`

## Purpose of `plan/`

The `plan/` subdirectory implements the `/plan` CLI command, which enables a "plan mode" for a permission-aware CLI tool. This command allows users to:

1. **Enable plan mode** — Enter a special mode where operations are tracked without being executed
2. **View the current plan** — Display all pending operations that have been queued
3. **Open the plan in an editor** — Use the `/plan open` subcommand to edit the plan file externally

## Contents Overview

The `plan/` directory contains two files that follow the standard command pattern used throughout the CLI:

| File | Role | Description |
|------|------|-------------|
| `index.ts` | Command registration | Exports the command definition object with metadata (name, description, type) and lazy-loading configuration |
| `plan.tsx` | Command implementation | Contains the `PlanDisplay` component for rendering UI and the `call` async function as the command entry point |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                    CLI Entry Point                      │
│                    (user types /plan)                    │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              index.ts (command definition)               │
│  ┌───────────────────────────────────────────────────┐  │
│  │ name: "plan"                                      │  │
│  │ description: "Enable plan mode..."                │  │
│  │ load: () => import('./plan.tsx')                  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────────┘
                      │ dynamic import
                      ▼
┌─────────────────────────────────────────────────────────┐
│               plan.tsx (implementation)                  │
│                                                          │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │ PlanDisplay      │    │ call(onDone, context,    │  │
│  │ (Ink component)  │    │            args)         │  │
│  │                  │    │                          │  │
│  │ - Shows header   │    │ - Check current mode     │  │
│  │ - Shows file path│    │ - Enable plan mode OR    │  │
│  │ - Renders content│    │ - Display plan OR        │  │
│  │ - Shows editor   │    │ - Open in external editor│  │
│  │   hint           │    │                          │  │
│  └──────────────────┘    └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Relationship summary:**

1. **`index.ts`** acts as a lightweight registration layer that defines command metadata and defers the heavy implementation loading
2. **`plan.tsx`** contains the actual command logic, UI rendering, and state management
3. The dynamic import pattern (`load: () => import('./plan.tsx')`) ensures the React/Ink runtime is only loaded when the command is actually invoked

## Key Takeaways

- **Separation of concerns**: Command metadata (index.ts) is separated from implementation details (plan.tsx), enabling lazy loading and faster CLI startup
- **Permission system integration**: The command integrates deeply with the app's permission system, using `prepareContextForPlanMode()` and `applyPermissionUpdate()` to manage mode transitions
- **Ink-based rendering**: Uses `renderToString()` from a static render utility to convert Ink/React components into string output for the CLI
- **React Compiler optimization**: The `PlanDisplay` component uses the React compiler runtime with manual memoization via `_c` sentinel symbols for performance
- **Editor integration**: Supports opening the plan file in the user's preferred terminal editor (Detected via `getExternalEditor()`)
- **Conditional behavior**: The same command entry point handles three scenarios based on current state:
  - Not in plan mode → enable it
  - In plan mode without "open" → display current plan
  - In plan mode with "open" → open file in external editor
