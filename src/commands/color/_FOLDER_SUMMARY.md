# Summary of `commands/color/`

## Purpose of `color/`

Provides a CLI command that allows users to set or reset the visual color of their agent session. The command validates input, persists the color across sessions via transcript storage, and updates the UI state immediately.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Command entry point with metadata and lazy-loading configuration |
| `color.ts` | Core implementation: validation, persistence, state updates |

## How Files Relate to Each Other

```
┌─────────────────────────────────────┐
│         index.ts (entry)            │
│  - Defines command metadata         │
│  - Uses lazy import('./color.js')   │
└────────────────┬────────────────────┘
                 │ dynamic import
                 ▼
┌─────────────────────────────────────┐
│         color.ts (implementation)   │
│  - Validates color/resets           │
│  - Checks teammate restrictions     │
│  - Persists to session storage      │
│  - Updates AppState                 │
└─────────────────────────────────────┘
```

**Flow**: User invokes `color` → `index.ts` lazy-loads `color.ts` → `color.ts` processes the command, saves the color, and triggers UI update via `onDone` callback.

## Key Takeaways

- **Lazy loading**: The `index.ts` defers loading of the heavy implementation to reduce CLI startup time.
- **Reset sentinel**: Uses string `"default"` (not empty/undefined) for cross-session persistence of reset colors.
- **Teammate restriction**: Swarm teammates cannot set their own color—only the team leader can assign colors.
- **Dual storage**: Color is written to both the transcript file (for persistence) and the app state (for immediate effect).
- **Validation chain**: Input flows through teammate check → empty args check → reset alias check → color whitelist validation before persistence.
