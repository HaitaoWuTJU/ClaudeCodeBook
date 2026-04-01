# Summary of `commands/effort/`

## Purpose of `effort/`

The `effort/` directory implements the `/effort` command, which allows users to query and set the effort level that controls how thoroughly the AI model approaches tasks. Effort levels range from quick implementations to maximum capability with deep reasoning.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command configuration object (name, description, lazy-load entry) |
| `effort.ts` | Full command implementation (logic, React components, analytics) |

## How Files Relate to Each Other

```
index.ts (command config)
    │
    │ loads via lazy import
    ▼
effort.ts (implementation)
    │
    ├── setEffortValue()     → writes to userSettings, logs analytics
    ├── showCurrentEffort()  → reads appState + env vars, returns status message
    ├── unsetEffortLevel()   → clears persisted effort setting
    ├── executeEffort()      → normalizes args, routes to set/unset
    ├── ShowCurrentEffort    → React component (displays current effort)
    ├── ApplyEffortAndClose  → React component (applies effort, closes)
    └── call()               → async entry point invoked by command system
```

The `index.ts` file acts as the **configuration layer**, while `effort.ts` contains the **business logic and UI components**. When the `/effort` command is invoked, the system loads `effort.ts` via the lazy import in `index.ts`.

## Key Takeaways

- **Valid effort levels**: `low`, `medium`, `high`, `max` (Opus 4.6 only), `auto`
- **Persistence strategy**: Session-only effort values (e.g., "max" on unsupported models) are displayed with a suffix; persistent values are written to user settings
- **Environment override**: The `CLAUDE_CODE_EFFORT_LEVEL` env var takes precedence and triggers informational messages when conflicting with user intent
- **Analytics integration**: All effort changes (including "auto") are tracked as `tengu_effort_command` events
- **React compiler compatibility**: Components use `_c` from `react/compiler-runtime` with optimized selector patterns
- **Lazy loading**: The actual implementation is not loaded until the command is invoked, improving startup performance
