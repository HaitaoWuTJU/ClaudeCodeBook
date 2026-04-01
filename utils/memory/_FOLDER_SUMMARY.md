# Summary of `utils/memory/`

## Purpose of `memory/`
This directory defines type-safe memory type identifiers used throughout the codebase, providing both a runtime constant array and a corresponding TypeScript union type.

## Contents Overview

| File | Description |
|------|-------------|
| `types.ts` | Defines `MEMORY_TYPE_VALUES` tuple and `MemoryType` union type with conditional 'TeamMem' support |

## How Files Relate to Each Other

```
memory/
└── types.ts
    ├── Exports: MEMORY_TYPE_VALUES, MemoryType
    └── Imports: feature from bun:bundle (Bun runtime)
```

- **`types.ts`** is the single source of truth for memory type definitions
- Both the runtime array (`MEMORY_TYPE_VALUES`) and the type (`MemoryType`) are kept in sync automatically via `as const` inference

## Key Takeaways

- **Feature-gated type**: `'TeamMem'` is conditionally included based on `feature('TEAMMEM')`, allowing incremental rollout
- **Type-safe enums**: Using `(typeof MEMORY_TYPE_VALUES)[number]` creates a union type from the array values, guaranteeing type–value alignment
- **Bun-specific**: Relies on `bun:bundle` for feature flag resolution at build time
- **Exported values**: `MEMORY_TYPE_VALUES` and `MemoryType` are available for import by any module needing to validate or switch on memory types
