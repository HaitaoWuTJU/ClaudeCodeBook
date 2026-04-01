# Summary of `commands/resume/`

## Purpose of `resume/`

Provides a command that allows users to resume previous AI coding conversations. The directory contains both the command registration metadata and the interactive implementation for selecting and resuming sessions.

## Contents Overview

| File | Role |
|------|------|
| `resume.ts` | Command registration — defines the command name, aliases, description, and lazy-loads the implementation |
| `resume.tsx` | Command implementation — React component with interactive picker UI, session lookup, filtering, and resumption logic |

## How Files Relate to Each Other

```
resume.ts (Command Config)
    │
    └──► lazy-imports ──► resume.tsx (Implementation)
                               │
                               ├──► LogSelector component (UI picker)
                               ├──► Session storage utilities (reads log files)
                               ├──► UUID validation (input parsing)
                               ├──► Worktree path detection (git support)
                               └──► Clipboard utilities (cross-project resume)
```

1. **`resume.ts`** is the entry point registered with the command framework. When a user types `/resume`, the framework uses this configuration for help text and aliases.

2. The `load` function performs a dynamic import of **`resume.tsx`** only when the command is actually invoked, keeping initial load times fast.

3. **`resume.tsx`** contains the full implementation:
   - Loads session logs from filesystem
   - Filters out the current session and sidechains
   - Displays an interactive picker or resolves a direct session ID
   - Triggers resumption via `onResume()`

## Key Takeaways

- **Lazy loading**: The heavy React component is never loaded unless the user actually runs the `/resume` command
- **Multiple lookup methods**: Supports direct UUID input, custom title search, and interactive picker
- **Cross-project awareness**: Can resume sessions from different repositories by copying commands to clipboard
- **Git worktree support**: Properly detects and loads logs from worktrees of the same repository
- **Fallback handling**: If enriched session data is unavailable, falls back to direct file lookups
