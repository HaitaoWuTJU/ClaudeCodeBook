# Summary of `commands/diff/`

## Purpose of `diff/`

The `diff/` directory implements a "diff" command that allows users to view uncommitted changes and per-turn diffs. It follows a lazy-loading pattern where the command metadata is separated from the implementation, enabling code splitting.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Command loader — exports metadata (`name`, `type`, `description`) and a dynamic import to `./diff.js` |
| `diff.tsx` | Command implementation — exports an async `call` function that renders `DiffDialog` with `messages` and `onDone` |

## How Files Relate to Each Other

```
index.ts (loader)
    │
    └── dynamically imports ──► diff.tsx (implementation)
                                      │
                                      └── dynamically imports ──► DiffDialog component
```

1. When the "diff" command is invoked, `index.ts` loads first as the entry point
2. Its `load()` function triggers dynamic import of `diff.tsx`
3. The `call` function in `diff.tsx` further lazy-loads the `DiffDialog` component
4. `DiffDialog` receives `context.messages` and the `onDone` callback as props

## Key Takeaways

- **Lazy Loading**: Both the command loader and the dialog component use `dynamic import()` for code splitting, keeping initial bundle sizes small
- **Two-Layer Code Splitting**: The directory uses two levels of lazy loading (index → diff → DiffDialog)
- **Async Command Pattern**: Commands return async functions that resolve to JSX, following a specific `LocalJSXCommandCall` signature
- **Separation of Concerns**: Metadata/registration logic is cleanly separated from rendering logic
