# Summary of `commands/remote-env/`

## Purpose of `remote-env/`

This directory implements the CLI command for configuring the default remote environment used during teleport sessions. It provides a user-facing dialog that allows subscribers to select or configure their remote environment settings.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command configuration and loader — defines metadata, availability checks, and lazy-loads the implementation |
| `remote-env.tsx` | Command implementation — renders the `RemoteEnvironmentDialog` UI component |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLI Entry Point                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  index.ts — Command Configuration                                │
│  ├── Checks: isClaudeAISubister() + isPolicyAllowed(...)        │
│  ├── Exports: type, name, description, isEnabled, isHidden       │
│  └── load() ─────► dynamic import('./remote-env.tsx')           │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  remote-env.tsx — Command Implementation                         │
│  └── call(onDone) ──► <RemoteEnvironmentDialog onDone={...} /> │
└─────────────────────────────────────────────────────────────────┘
```

**Execution Flow:**

1. CLI loads `index.ts` and evaluates `isEnabled()` — returns `false` if user lacks subscription or policy permission
2. When command is invoked, `load()` performs dynamic import of `remote-env.tsx`
3. The exported `call(onDone)` function renders the dialog component
4. User interacts with `RemoteEnvironmentDialog` and completes configuration
5. `onDone` callback signals command completion

## Key Takeaways

- **Access Control**: The command is gated behind two independent checks — active Claude AI subscription AND the `allow_remote_sessions` policy flag
- **Lazy Loading**: The actual UI implementation is not bundled at startup; it's only loaded when the command executes
- **Lazy Evaluation**: The `isHidden` getter recalculates availability on each access rather than caching the result
- **JSX Command Pattern**: Uses the `local-jsx` command type with an async `call()` function that returns `Promise<React.ReactNode>`
