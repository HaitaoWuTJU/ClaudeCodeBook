# Summary of `commands/config/`

## Purpose of `config/`

Provides a command that opens the application settings panel with the "Config" tab selected by default. Serves as the entry point for users to configure application settings via the "config" or "settings" command.

## Contents Overview

| File | Role | Description |
|------|------|-------------|
| `index.ts` | Command definition | Declares command metadata (name, aliases, type) and lazy-loads the implementation |
| `config.jsx` | Command implementation | Actual logic that renders the `Settings` component |

## How Files Relate to Each Other

```
index.ts (command definition)
    │
    │ exports 'config' command with type: 'local-jsx'
    │ provides lazy load function: () => import('./config.jsx')
    │
    ↓
config.jsx (command implementation)
    │
    │ imports Settings component and CommandCall type
    │ exports async call() function
    │
    ↓
Settings component (external)
    │ receives: onClose, context, defaultTab="Config"
    └─► renders the Config settings panel
```

## Key Takeaways

- **Two-file pattern**: Separate configuration metadata (`index.ts`) from implementation (`config.jsx`) enables lazy loading
- **JSX command type**: Uses `local-jsx` type, meaning the command returns a React component rather than plain data
- **Hardcoded default tab**: The Settings component always opens to "Config" tab—there's no parameter to change this from the command side
- **Minimal abstraction**: This directory is thin—most logic lives in the `Settings` component itself; this only acts as a wiring layer
