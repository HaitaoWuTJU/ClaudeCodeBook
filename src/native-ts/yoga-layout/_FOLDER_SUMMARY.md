# Summary of `native-ts/yoga-layout/`

## Purpose of `native-ts/yoga-layout/`

This directory contains a **pure TypeScript port of Facebook's Yoga flexbox layout engine** — the same library behind `yoga-layout/load` but implemented entirely in JavaScript without WASM or native bindings. The goal is to compute CSS Flexbox layouts (positions, sizes, margins, padding) for UI node trees, providing a portable JavaScript implementation that matches the upstream API and behavior exactly.

## Contents Overview

```
native-ts/yoga-layout/
├── enums.ts   — 16 enum types + their union type aliases
└── index.ts   — Core layout engine implementation (~83KB)
```

### `enums.ts`
Declares 16 enum groups as `const` objects (not TypeScript's `enum` keyword), each paired with a derived union type. These cover all Yoga configuration dimensions:

- **Alignment**: `Align`, `Justify`, `Gutter`
- **Sizing**: `Dimension`, `Unit`, `BoxSizing`
- **Direction**: `Direction`, `FlexDirection`, `Wrap`
- **Positioning**: `Edge`, `PositionType`, `Overflow`, `Display`
- **Measurement**: `MeasureMode`, `Errata`, `ExperimentalFeature`

### `index.ts`
The main engine, organized into these logical layers:

| Layer | What it does |
|-------|-------------|
| **Types** | `Value`, `Layout`, `Style`, `Config`, `Node` |
| **Edge resolution** | `resolveEdge()`, `resolveEdges4Into()` — maps Yoga's 9-edge model to 4 physical edges |
| **Axis helpers** | `isRow()`, `isReverse()`, `leadingEdge()`, `trailingEdge()`, `crossAxis()` |
| **Config & Node** | `Config` class, `Node` class with all style/layout/children storage |
| **Core layout** | `layoutNode()`, `computeFlexBasis()`, `resolveFlexibleLengths()` |
| **Baseline** | `calculateBaseline()`, `isBaselineLayout()` |
| **Utilities** | `collectLayoutChildren()`, `roundLayout()`, `resolveGap()`, `boundAxis()` |
| **API entry** | `loadYoga()` + `YOGA_INSTANCE` singleton matching `yoga-layout/load` |

## How Files Relate to Each Other

```
enums.ts                    index.ts
─────────────              ────────────
Align ───────────────────→ Align (imported)
BoxSizing ───────────────→ BoxSizing
Dimension ────────────────→ Dimension
Direction ────────────────→ Direction
Display ──────────────────→ Display
Edge ─────────────────────→ Edge (used heavily by resolveEdge, margin/padding helpers)
Errata ───────────────────→ Errata
ExperimentalFeature ──────→ ExperimentalFeature
FlexDirection ────────────→ FlexDirection (used in axis helpers)
Gutter ───────────────────→ Gutter
Justify ──────────────────→ Justify
MeasureMode ──────────────→ MeasureMode
Overflow ─────────────────→ Overflow
PositionType ─────────────→ PositionType
Unit ─────────────────────→ Unit (core to Value type, resolveValue)
Wrap ─────────────────────→ Wrap
```

**Dependency direction**: `enums.ts` is a pure leaf — no imports, only exports. `index.ts` imports all enums and builds the layout algorithm on top of them.

## Key Takeaways

### Why a pure-TS port?
Provides a JavaScript-native alternative to the WASM/C++ yoga-layout without requiring native bindings or build toolchain integration. Callers get the same API (`Yoga.Config`, `Yoga.Node`, `loadYoga()`) with identical layout behavior.

### Multi-pass flex resolution
`resolveFlexibleLengths()` iterates until stable, freezing violated children per CSS Flexbox §9.7. This is the core of flex layout — handling `flex-grow`, `flex-shrink`, and min/max constraint violations.

### Aggressive caching
The engine maintains three cache layers per node to avoid redundant computation:
1. **Single-entry layout cache** — immediate hit when inputs are identical
2. **Multi-entry layout cache** — 4 slots × 8 input combinations for cross-pass reuse
3. **Basis cache** — skips remeasure of children with `measureFunc` (e.g., text) when dimensions haven't changed

A global `_generation` counter invalidates stale caches across mount/unmount cycles.

### Display:contents flattening
`collectLayoutChildren()` recursively lifts children of `display:contents` nodes to the grandparent level — the contents node's own box is elided per CSS spec.

### Pixel-perfect rounding
`roundLayout()` implements `YGRoundValueToPixelGrid` behavior from upstream's `PixelGrid.cpp`. Text nodes (those with `measureFunc`) get floor positions and ceil widths to prevent fractional-pixel gaps in wrapped text. Without this, justify-content:center and space-evenly produce off-by-one alignment.

### Fast-path flags
`Node` stores `_hasAutoMargin`, `_hasPosition`, `_hasPadding`, `_hasBorder`, `_hasMargin` at construction and after style updates. These skip expensive property reads in hot layout paths.
