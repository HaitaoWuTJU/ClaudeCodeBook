# Summary of `components/HighlightedCode/`

## Purpose of `HighlightedCode/`

The `HighlightedCode/` directory provides React components for rendering syntax-highlighted code snippets in a terminal (CLI) environment. It uses **Ink** (React for CLIs) and **highlight.js** to colorize source code with ANSI escape sequences, offering a fallback mechanism when the highlighter is unavailable or a language is unsupported.

---

## Contents Overview

| File | Role |
|---|---|
| **`Fallback.tsx`** | Main highlighted code renderer. Handles language inference from file extensions, tab-to-space conversion, async highlight.js loading via React Suspense, and a module-level LRU cache (500 entries) to avoid re-highlighting on virtual-scroll remounts. Falls back to "markdown" language for unknown or unsupported languages. |

---

## How Files Relate to Each Other

```
HighlightedCode/
└── Fallback.tsx
```

The `Fallback.tsx` component is the sole file in this directory. It integrates several utilities from the broader codebase:

- **`utils/cliHighlight.js`** — `getCliHighlightPromise()` provides an async promise for loading highlight.js on demand, enabling React Suspense integration.
- **`utils/hash.js`** — `hashPair()` generates cache keys from a `(language, code)` pair to store/retrieve highlighted results.
- **`utils/file.js`** — `convertLeadingTabsToSpaces()` normalizes code indentation before highlighting.
- **`ink.js`** — `Ansi` and `Text` components render ANSI-colored terminal output with optional dimming.

---

## Key Takeaways

1. **Module-level LRU Cache** — The cache prevents expensive re-highlighting during virtual-scroll remounts. It caps at 500 entries and evicts the oldest entry on overflow, mitigating RSS growth (fixes #24180).

2. **Async + Suspense** — The component uses React's `use()` hook on an async highlight.js promise wrapped in Suspense, ensuring the library loads without blocking the render tree.

3. **Graceful Degradation** — If `skipColoring` is `true`, it renders plain code. If highlight.js doesn't support the inferred language, it falls back to `"markdown"` highlighting.

4. **Language Inference** — Language is extracted from the file extension (e.g., `.ts` → `typescript`) and passed to highlight.js.

5. **Tab Normalization** — Leading tabs in source code are converted to spaces to ensure consistent rendering across terminal widths.
