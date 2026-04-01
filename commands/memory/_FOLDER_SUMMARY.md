# Summary of `commands/memory/`

## Purpose of `memory/`

The `memory/` subdirectory implements a CLI command that allows users to view and edit Claude's memory files through an interactive dialog interface. It leverages lazy loading to separate the command registration (in `memory.ts`) from the actual implementation (in `memory.tsx`).

## Contents Overview

| File | Purpose |
|------|---------|
| `memory.ts` | Command definition/registration file — exports a `Command` object with lazy-loading via `() => import('./memory.js')` |
| `memory.tsx` | Full implementation — React/Ink component that renders a dialog for selecting memory files, creates directories/files as needed, and opens the selected file in the user's preferred editor |

## How Files Relate to Each Other

```
memory.ts (lazy loader)
    │
    └── dynamically imports ──► memory.tsx (implementation)
                                    │
                                    ├── reads: MemoryFileSelector, getMemoryFiles(), $EDITOR/$VISUAL
                                    ├── writes: mkdir (dirs), writeFile (files)
                                    └── transforms: Opens file in external editor
```

1. **`memory.ts`** serves as a lightweight entry point that registers the command without loading the heavy implementation immediately
2. **`memory.tsx`** contains the full React component tree with `MemoryCommand` as the root
3. The `call` function in `memory.tsx` pre-primes memory caches before rendering, ensuring smooth UX

## Key Takeaways

- **Lazy loading pattern**: The command implementation is loaded on-demand, keeping the initial bundle small
- **Interactive CLI experience**: Uses Ink (React for terminals) to render an interactive dialog with file selection and help links
- **Editor integration**: Respects user environment (`$VISUAL` → `$EDITOR` → default) for opening files
- **Safe file operations**: Uses `flag: 'wx'` for atomic file creation that won't overwrite existing data
- **Themed UI**: Uses `color="remember"` for consistent styling with the memory feature
- **Error resilience**: Handles `EEXIST` errors gracefully when initializing memory files
