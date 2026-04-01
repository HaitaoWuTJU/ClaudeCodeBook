# Summary of `native-ts/file-index/`

## Purpose of `file-index/`

This directory provides a pure TypeScript implementation of a high-performance fuzzy file search engine, originally ported from the Rust NAPI module `nucleo`. The core responsibility is to index a list of file paths and enable fast fuzzy matching against user queries, returning ranked results sorted by match quality. This is designed to power UI features such as file pickers, command palettes, and quick-open dialogs.

## Contents Overview

| File | Description |
|------|-------------|
| `file-index.ts` | **Core module** containing the entire implementation: the `FileIndex` class, `search()` function, scoring constants, helper utilities, and all algorithmic logic for indexing and querying. |

## How Files Relate to Each Other

The implementation is self-contained in a single file. The relationship is purely internal:

```
file-index.ts
├── FileIndex class
│   ├── loadFromFileList() ──► resetArrays() + indexPath()
│   ├── loadFromFileListAsync() ──► resetArrays() + indexPath() + yieldToEventLoop()
│   ├── indexPath() ──► updates lowerPaths[], charBits[], pathLens[]
│   └── search() ──► uses charBits for O(1) reject, posBuf[] for matches
├── search() ──► uses scoreBonusAt(), computes top-k results
├── computeTopLevelEntries() ──► populates topLevelCache[] for empty queries
├── yieldToEventLoop() ──► called during async loading
└── Helper functions (isBoundary, isLower, isUpper, scoreBonusAt)
```

All data flows through `FileIndex` internal state (`paths`, `lowerPaths`, `charBits`, `pathLens`, `topLevelCache`, `posBuf`), and the `search()` method orchestrates the full scoring pipeline.

## Key Takeaways

- **Zero external dependencies** — uses only built-in JavaScript/Web APIs, making it portable and tree-shakeable.
- **O(1) path rejection via bitmaps** — each path stores a 26-bit bitmap of lowercase characters present; queries build a similar bitmap and use bitwise AND to eliminate non-matching paths instantly.
- **Top-k selection over full sort** — maintains a sorted array of the best `limit` results during traversal, achieving O(limit × log limit) instead of O(n log n) for sorting.
- **Chunked async loading** — `loadFromFileListAsync` yields to the event loop every ~4ms in ~256-entry chunks, keeping the main thread responsive during large index builds.
- **fzf/nucleo scoring semantics** — implements boundary bonuses (8pts), camelCase bonuses (6pts), consecutive match bonuses (4pts), and gap penalties (3 + 1/char) for human-friendly ranking.
- **Test file deprioritization** — paths containing `"test"` receive a 1.05× score penalty to push them lower in results.
- **SIMD-accelerated** — `indexOf` is leveraged for the main character scan; modern JS engines vectorize this operation for significant speedups on long strings.
- **Progressive queryability** — the async loader returns a `queryable` promise that resolves as soon as the index is partially built, allowing searches to begin before full completion.
