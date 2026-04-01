# Summary of `moreright/`

## Purpose of `moreright/`

The `moreright/` directory contains the `useMoreRight` hook, a feature-related hook for a chat/Messages UI system. It provides "More Right" functionality—likely contextual menu, additional options, or extended controls for message handling. This particular file is a **stub implementation** designed for external builds where the internal implementation cannot be properly imported.

## Contents Overview

| File | Description |
|------|-------------|
| `useMoreRight.tsx` | Stub implementation of `useMoreRight` hook with no-op methods |

The `useMoreRight` hook returns an object with three methods:

- `onBeforeQuery()` — Pre-query callback (stubbed to always return `true`)
- `onTurnComplete()` — Turn completion callback (stubbed to no-op)
- `render()` — UI rendering method (stubbed to return `null`)

## How Files Relate to Each Other

```
moreright/
└── useMoreRight.tsx  (standalone stub, no internal dependencies)
```

The directory is minimal and contains only `useMoreRight.tsx`. This stub exists independently with **no relative imports** or dependencies, making it suitable for external build environments where internal paths (like `../types/`) cannot resolve correctly.

## Key Takeaways

1. **Stub Pattern**: This is a placeholder implementation—actual logic is handled internally and not exposed to external builds.

2. **Type Safety**: The stub satisfies TypeScript type checking while providing no runtime functionality, allowing external projects to compile without import errors.

3. **Feature Hooks Architecture**: The codebase separates feature hooks into dedicated directories, with `useMoreRight` designed to handle interactive behaviors (pre-query checks, turn completion, rendering).

4. **Build Environment Awareness**: The comments explicitly note that TypeScript sees this file from `scripts/external-stubs/src/moreright/` before overlay, indicating a build system that redirects imports in external contexts.
