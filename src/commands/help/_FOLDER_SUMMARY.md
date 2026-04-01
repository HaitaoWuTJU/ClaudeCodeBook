# Summary of `commands/help/`

## Purpose of `help/`

The `help/` subdirectory implements a **"help" command** that displays available commands in a user interface. It follows a lazy-loading pattern where the actual help component is only loaded when the command is invoked.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command configuration and lazy loader |
| `help.tsx` | Command handler that renders the `HelpV2` component |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                        Command Loader                       │
└─────────────────────────┬───────────────────────────────────┘
                          │ loads on demand
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  index.ts - Command Configuration                           │
│  ├── type: 'local-jsx'                                      │
│  └── import(): lazy loads help.tsx                          │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  help.tsx - Command Handler (call function)                  │
│  ├── Receives: onDone callback + commands from options      │
│  └── Returns: <HelpV2 commands={...} onClose={onDone} />   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  HelpV2 Component (external)                                │
│  └── Renders the help UI                                    │
└─────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Lazy Loading Pattern**: The help component is not bundled until needed, improving initial load times
2. **Command Interface**: Both files adhere to a `Command` interface with `type: 'local-jsx'`
3. **Callback Bridge**: The `help.tsx` handler bridges the command API's `onDone` callback to the component's `onClose` prop
4. **TypeScript + JSX**: The `.tsx` extension indicates TypeScript with JSX support for React component rendering
5. **Props Flow**: Commands are passed from options → help handler → HelpV2 component
