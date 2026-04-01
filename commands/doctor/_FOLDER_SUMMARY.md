# Summary of `commands/doctor/`

## Purpose of `doctor/`

Provides a diagnostic "doctor" command for the CLI that verifies and displays the status of the Claude Code installation, configuration, and environment. The command renders a UI screen (`Doctor`) that shows diagnostics information to the user.

## Contents Overview

The directory contains two files working together:

| File | Role |
|------|------|
| `index.ts` | Command entry point: defines metadata, lazy-loads implementation |
| `doctor.tsx` | Command implementation: renders the `Doctor` screen component |

## How Files Relate to Each Other

```
Command invoked by user
         │
         ▼
┌─────────────────────────┐
│      index.ts           │
│  • Checks DISABLE_* env │
│  • Lazy imports doctor  │
└────────────┬────────────┘
             │
             ▼ (dynamic import)
┌─────────────────────────┐
│      doctor.tsx         │
│  • Type: local-jsx      │
│  • Calls Doctor component│
│  • Returns Promise<JSX> │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   screens/Doctor.js     │
│  • Renders diagnostic UI│
└─────────────────────────┘
```

**Data flow**: `index.ts` acts as a thin wrapper providing lazy-loading and a feature flag. When invoked, it loads `doctor.tsx` which creates and returns the `Doctor` component wrapped in a Promise. The `Doctor` screen receives `onDone` to notify when diagnostics are complete.

## Key Takeaways

- **Lazy loading**: Implementation is split into a separate file and dynamically imported to optimize CLI startup time
- **Feature flag**: The command can be disabled via `DISABLE_DOCTOR_COMMAND` environment variable
- **Command type**: Uses `local-jsx` type, indicating this command renders JSX locally rather than streaming output
- **Bridge pattern**: `doctor.tsx` adapts the command infrastructure to React by returning `Promise.resolve(<Doctor />)`
- **Unused parameters**: `_context` and `_args` are intentionally ignored, suggesting optional command metadata that isn't needed for diagnostics
