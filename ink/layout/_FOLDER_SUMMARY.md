# Summary of `ink/layout/`

## Purpose of `layout/`

Provides the **layout engine abstraction layer** for the Ink terminal UI library, enabling flexbox-based layout computation for React-like terminal interfaces. The module separates concerns between interface definitions, concrete implementation, and geometric utilities.

## Contents Overview

| File | Role | Size |
|------|------|------|
| `node.ts` | Interface contract & type constants | ~1.7 KB |
| `yoga.ts` | Yoga layout engine adapter (implementation) | ~5.3 KB |
| `geometry.ts` | Geometric primitives & utilities | ~2.5 KB |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                        layout/                               │
│                                                             │
│  ┌──────────────┐    ┌──────────────────┐   ┌───────────┐  │
│  │   geometry   │    │      node        │   │   yoga    │  │
│  │              │    │                  │   │           │  │
│  │ Types:       │    │ Types:           │   │ Implements│  │
│  │ • Point      │    │ • LayoutEdge     │   │ LayoutNode│  │
│  │ • Size       │    │ • LayoutGutter   │   │ interface │  │
│  │ • Rectangle  │    │ • LayoutDisplay  │   │           │  │
│  │ • Edges      │    │ • LayoutAlign    │   │ Wraps:    │  │
│  │              │    │ • LayoutJustify  │   │ Yoga Node │  │
│  │ Functions:   │    │ • LayoutWrap     │   │           │  │
│  │ • edges()    │    │ • LayoutPosition │   │ Maps enums│  │
│  │ • unionRect  │    │ • LayoutOverflow │   │ to native │  │
│  │ • clampRect  │    │                  │   │ Yoga types│  │
│  │ • withinBnds │    │ Interface:       │   │           │  │
│  │ • clamp()    │    │ LayoutNode       │   │           │  │
│  │ • addEdges   │    │                  │   │           │  │
│  └──────┬───────┘    └────────┬─────────┘   └─────┬─────┘  │
│         │                     │                    │        │
│         └─────────────────────┼────────────────────┘        │
│                               │                              │
│         Internal consumers use types & interface             │
│         from this module for layout calculations             │
└─────────────────────────────────────────────────────────────┘
```

**Dependency chain:**

1. **`geometry.ts`** ← Standalone (no dependencies)
   - Defines reusable geometric types used by layout components
   
2. **`node.ts`** ← Standalone (no dependencies)
   - Exports `LayoutNode` interface and layout-related enum types
   - No coupling to geometry utilities
   
3. **`yoga.ts`** ← Depends on `node.ts` + external `yoga-layout`
   - Imports `LayoutNode` interface from `node.ts`
   - Implements the interface by wrapping native Yoga nodes
   - Maps internal enum types to native Yoga enum values

## Key Takeaways

- **Abstraction pattern**: The `LayoutNode` interface in `node.ts` decouples consumers from the Yoga implementation, allowing the layout engine to be swapped (e.g., CSS Box, native Rust bindings) without changing calling code.

- **No external runtime dependencies**: The TypeScript yoga-layout port is synchronous and self-contained—`yoga-layout` is a dev/build dependency rather than a runtime import.

- **Adapters handle enum translation**: `EDGE_MAP` and `GUTTER_MAP` in `yoga.ts` are simple `Record` lookups that convert internal enum values to native Yoga enum values, keeping the adapter thin.

- **Measure callback adaptation**: Custom measure functions are adapted via a closure that converts Yoga's `MeasureMode` enum to internal `LayoutMeasureMode` before invoking the user-provided callback.

- **Explicit memory management**: Native Yoga nodes are explicitly freed via `free()`/`freeRecursive()`, indicating manual resource control rather than relying on garbage collection for native interop.

- **Geometry utilities are supplementary**: `geometry.ts` is optional for layout consumers—`LayoutNode` interface does not depend on it. It likely serves as helper utilities for coordinate/bounds manipulation during layout composition.
