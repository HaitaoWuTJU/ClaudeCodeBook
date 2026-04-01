# Summary of `native-ts/`

## Purpose of `native-ts/`

The `native-ts/` directory is a collection of **pure TypeScript/TypeScript ports** of performance-critical native modules used by Dprint's CLI. Instead of relying on Rust NAPI bindings or native WASM, these directories reimplement the same algorithms and APIs entirely in JavaScript — making them portable, dependency-free, and compatible with any JavaScript runtime. Each directory maps to a specific capability:

| Sub-directory | Responsibility |
|---------------|----------------|
| `file-index/` | Fuzzy file path search engine |
| `yoga-layout/` | CSS Flexbox layout computation |
| `color-diff/` | Syntax-highlighted ANSI diff rendering |

## Contents Overview

```
native-ts/
├── file-index/
│   ├── file-index.ts   — Fuzzy search engine (~600 lines)
│   └── deno.json       — Deno module config
├── yoga-layout/
│   ├── enums.ts        — 16 Yoga enum types + union aliases
│   ├── index.ts        — Layout engine implementation (~83KB)
│   └── deno.json       — Deno module config
└── color-diff/
    ├── colorDiff.ts    — Diff renderer with syntax highlighting (~700 lines)
    ├── mod.ts          — Module facade
    ├── deps.ts         — Dependency declarations
    └── deno.json       — Deno module config
```

## How Files Relate to Each Other

The three modules operate in distinct problem domains and share no runtime code, but they are **architecturally parallel** — each follows the same porting strategy:

```
native-ts/ (shared pattern)
│
├── {module}/deno.json       — Import maps route external deps to CDN URLs
├── {module}/deps.ts         — Deno: declares external dependency URLs
└── {module}/(main).ts       — Pure TS: uses deps + internal helpers
```

**Cross-module dependencies**: None. Each module is self-contained:
- `file-index/` uses only built-in JS APIs (no external deps)
- `yoga-layout/` uses only built-in JS APIs
- `color-diff/` uses `highlight.js` (loaded lazily via `require()`) and `diff` package

**Shared patterns across all three**:

1. **Deno compatibility layer** — `deno.json` + `deps.ts` route external packages (highlight.js, diff) through import maps, so the same source file works in both Node.js and Deno.
2. **Lazy loading** — Heavy dependencies (hljs ~50MB, highlight.js grammars) are loaded inside functions, not at module scope, to defer startup cost.
3. **API compatibility** — Each module exposes the same public API as its Rust counterpart:
   - `file-index.ts` → `FileIndex`, `search()` (matches `nucleo` Rust crate)
   - `yoga-layout/index.ts` → `loadYoga()` singleton (matches `yoga-layout/load`)
   - `color-diff/colorDiff.ts` → `getNativeModule()` (matches Rust `color_diff`)
4. **Internal test utilities** — `file-index.ts` and `colorDiff.ts` both export private `__test` objects for unit testing without exposing internals in the public API.

## Key Takeaways

1. **Replacement strategy**: These pure-TS ports serve as a drop-in fallback when native NAPI modules cannot be loaded (e.g., unsupported architectures, build failures), ensuring Dprint CLI works across all environments.

2. **No native bindings required**: Everything runs in pure JavaScript/WebAssembly (via hljs), enabling compatibility with Deno, browsers, and edge runtimes.

3. **Performance design patterns** used across all three modules:
   - **Precomputed data structures**: `file-index` uses 26-bit character bitmaps for O(1) rejection; `yoga-layout` caches computed layouts
   - **Top-k selection**: `file-index.search()` maintains only the best `limit` results instead of sorting all matches
   - **Lazy initialization**: `color-diff` defers hljs loading; `yoga-layout` uses a generation counter for cache invalidation

4. **Each module is a 1:1 semantic port**: The algorithms, scoring rules, and layout rules match the upstream Rust/Native implementations exactly — not approximations. This is confirmed by:
   - `file-index`: Same fzf/nucleo scoring formula (boundary 8pts, camel 6pts, consecutive 4pts, gap penalty 3 + 1/char)
   - `yoga-layout`: Same multi-pass flex resolution, pixel rounding rules, display:contents flattening, cache generation mechanism
   - `color-diff`: Same scope-to-color tables (MONOKAI_SCOPES, GITHUB_SCOPES, ANSI_SCOPES) reverse-engineered from Rust output

5. **Deno-first architecture**: The `deno.json` import maps allow the same source to be used in Deno without changes, with Node.js compatibility achieved through different resolution behavior.
