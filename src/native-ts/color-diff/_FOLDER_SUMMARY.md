# Summary of `native-ts/color-diff/`

## Purpose of `color-diff/`

The `color-diff/` directory contains a **pure TypeScript port** of a Rust color-diff module that provides syntax-highlighted diff rendering with ANSI color codes for terminal output. Its primary goal is to replace the Rust `syntect+bat` stack with JavaScript/TypeScript equivalents (`highlight.js` for syntax highlighting and the `diff` npm package for word-level diffing) while maintaining **API compatibility** with the original native module.

## Contents Overview

| File | Role |
|------|------|
| `colorDiff.ts` | **Main implementation** — the entire module (~700 lines). Exports `ColorDiff`, `ColorFile`, `getNativeModule()`, `getSyntaxTheme()`, and a `__test` object for internal utilities. |
| `mod.ts` | Module facade — re-exports the public API from `colorDiff.ts` |
| `deno.json` | Deno module configuration (import maps, tasks) |
| `deps.ts` | External dependency declarations (placeholder for Deno compatibility) |

The real logic lives entirely in `colorDiff.ts`, which can be consumed either:

- **Directly** via `import { getNativeModule } from './colorDiff.ts'` (TypeScript/JS path)
- **As a facade** via `import { ... } from './mod.ts'` (Deno-style module aliasing)

## How Files Relate to Each Other

```
mod.ts
 └── re-exports everything from colorDiff.ts
       ├── lazy-loads highlight.js (deferred ~190+ grammars, ~50MB)
       ├── uses diff (diffArrays) for word-level diffing
       ├── uses internal ink/stringWidth.js for wrapping
       └── uses internal utils/log.js for error reporting

deps.ts (Deno)
 └── declares external dependencies (placeholder)
       └── for Deno compatibility shim

deno.json
 └── import maps: maps 'highlight.js' → actual URL
       └── tasks: build/test scripts
```

**Flow**: `mod.ts` → `colorDiff.ts` → `highlight.js` / `diff` (external) + internal utils.

## Key Takeaways

1. **Lazy-loaded heavy dependency**: Highlight.js (~190+ language grammars, ~50MB) is loaded via `require()` inside a function to defer the ~100–200ms startup cost until first render. This prevents GC-pressure timeouts in CI environments.

2. **Syntect parity via measured scopes**: Scope-to-color tables (`MONOKAI_SCOPES`, `GITHUB_SCOPES`, `ANSI_SCOPES`) were reverse-engineered from the Rust module's actual syntect output. This ensures visual output matches the native module even though hljs grammars differ.

3. **Semantic scope compensation**: hljs doesn't scope plain identifiers or operators (`=`, `:`). The code re-splits `keyword` tokens into `storage.type` when they match `STORAGE_KEYWORDS` (const, let, var, function, class, etc.) to improve color accuracy.

4. **Robust version resilience**: A `hasRootNode()` runtime type guard validates `result.emitter` shape to catch hljs version mismatches (`scope` vs `kind`, `_emitter` vs `emitter`) before silent fallback.

5. **Processing pipeline** (per line): marker parsing → word-diff via `diffArrays` + adjacent-pair matching → syntax highlighting via hljs → newline removal → background layering → word-wrap → marker prepend → line number pad → ANSI escape output.

6. **ANSI 256-color approximation**: `ansi256FromRgb()` picks the closer of grey-ramp or 6×6×6 color-cube index using Euclidean RGB distance — a direct port of Rust's `ansi_colours` crate.

7. **Word-diff threshold**: If changed characters exceed 40% of total line length, `wordDiffStrings()` returns empty ranges — no word highlighting for large changes.

8. **Module exports**: The public API (`ColorDiff`, `ColorFile`, `getNativeModule()`, `getSyntaxTheme()`) is exposed via a lazy-cached singleton through `getNativeModule()`. Internal test utilities are available via `__test` for unit testing.
