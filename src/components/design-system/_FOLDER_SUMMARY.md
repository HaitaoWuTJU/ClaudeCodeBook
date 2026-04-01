# Summary of `components/design-system/`

## Purpose of `design-system/`

The `design-system/` directory is a **terminal UI component library** built on [Ink](https://github.com/vadimdemedes/ink) (React for CLIs). It provides theme-aware, composable primitives that render styled terminal interfaces with support for dark/light/auto theming, keyboard navigation, and consistent design tokens. Think of it as the equivalent of a browser-based component library (like Material UI or Chakra UI), but purpose-built for command-line applications where React components emit ANSI escape codes instead of CSS.

## Contents Overview

The directory exports **8 files** organized around two concerns: **theme infrastructure** and **UI primitives**.

### Theme Infrastructure

| File | Role |
|------|------|
| `ThemeProvider.tsx` | Root context provider; manages dark/light/auto theme state, watches OS system theme via OSC 11, and exposes hooks for reading/setting/previewing themes. |
| `color.ts` | Low-level utility function that resolves theme color keys (e.g., `"warning"`, `"error"`) or raw color values (`#ff0000`, `rgb(0,0,0)`, `ansi256(...)`) into strings Ink can render. |

### UI Primitives (styled with theme colors)

| File | Purpose |
|------|---------|
| `ThemedText.tsx` | Text component with full Ink styling support (bold, italic, underline, strikethrough, inverse, wrap) and a `TextHoverColorContext` for propagating hover colors across component boundaries. |
| `ThemedBox.tsx` | Theme-aware `Box` wrapper; resolves all border/background color props from theme keys before delegating to Ink's `Box`. |
| `Divider.tsx` | Horizontal rule with optional centered title, auto-sizing to terminal width. |
| `Byline.tsx` | Inline separator component; joins arbitrary children with a dimmed middot (` · `) for rendering metadata lines (e.g., author · date). |
| `Dialog.tsx` | Modal-style container with title/subtitle, keyboard shortcut hints (Esc, Ctrl+C/D), and configurable cancel behavior (can be disabled to allow text inputs to receive those keys). |
| `FuzzyPicker.tsx` | Interactive fuzzy-search picker with keyboard navigation, optional item grouping, multi-select support, and customizable item rendering via render prop. |

## How Files Relate to Each Other

```
ThemeProvider.tsx
      │
      ├── Provides: useTheme(), useThemeSetting(), usePreviewTheme()
      │
      ▼
┌──────────────────────────────────────────────────────────────────┐
│                         UI Components                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ThemedText ──► ThemedBox ──► Dialog ──► FuzzyPicker             │
│      │              │             │             │               │
│      │              │             │             ▼               │
│      │              │             │        FuzzySearch          │
│      │              │             ▼                              │
│      │              │        Byline ◄── (inline metadata)       │
│      │              │                                            │
│      │              ▼                                            │
│      └──────►  Divider                                           │
│                                                                  │
│  All components share:                                            │
│  ├── color.ts (resolves theme keys → raw colors)                │
│  ├── ThemeProvider context (resolved currentTheme)              │
│  └── Ink (Box, Text, Newline, etc.)                             │
└──────────────────────────────────────────────────────────────────┘
```

**Dependency direction**: `ThemeProvider` stands alone at the top. Every UI primitive depends on either `ThemedText`, `ThemedBox`, or both — which in turn depend on `color.ts` and `ThemeProvider` for color resolution. Higher-level components like `Dialog` and `FuzzyPicker` compose the primitives together.

## Key Takeaways

1. **Theme-first architecture**: Every component that accepts a color prop can receive either a raw color value or a theme key. The `color.ts` utility handles both transparently, enabling a single source of truth for design tokens.

2. **React Compiler ready**: All components use `react/compiler-runtime` (`_c`) with `$` memoization arrays, indicating the codebase is compiled with Babel's React Compiler for automatic optimization.

3. **Ink-based rendering**: All primitives wrap Ink components (`Box`, `Text`, `Newline`, etc.), meaning the design system operates entirely in the terminal environment rather than the browser.

4. **Composition over configuration**: The system favors small, focused components (`ThemedText`, `ThemedBox`, `Byline`) that compose into larger patterns (`Dialog`, `FuzzyPicker`), following the React component composition model.

5. **Terminal-specific concerns**: Unlike web design systems, this library handles terminal-specific realities — OSC 11 system theme detection, keyboard shortcut handling, ANSI color codes, and fixed-width text rendering (via `useTerminalSize` in `Divider`).

6. **Preview pattern for theme picker**: `ThemeProvider` implements a three-phase preview workflow (`setPreviewTheme` → `savePreview`/`cancelPreview`) to show temporary theme changes without persisting until confirmed.
