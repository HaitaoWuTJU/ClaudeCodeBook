# Summary of `ink/components/`

# `ink/components/` Directory Summary

## Purpose

This directory contains the **core rendering primitives** for Ink — a React renderer for terminal/CLI applications. These components bridge the gap between React's component model and the terminal's raw I/O, escape sequences, and state management. Together, they form the foundation that any Ink application runs on.

## Contents Overview

| File | Role |
|------|------|
| `App.tsx` | **Root component** — orchestrates stdin/stdout, raw mode, keyboard/mouse parsing, text selection, error boundaries, and provides all major contexts |
| `AppContext.ts` | **Internal stdin context** — exposes `stdin`, `setRawMode()`, `eventEmitter`, `terminalQuerier`, and `exitOnCtrlC` to the component tree |
| `TerminalSizeContext.tsx` | **Terminal dimensions** — provides `columns` and `rows` via React Context |
| `TerminalFocusContext.tsx` | **Terminal focus tracking** — exposes `isTerminalFocused` and `terminalFocusState` via `useSyncExternalStore` |
| `TerminalSizeContext.tsx` (also) | **Type definition** — exports `TerminalSize` type alongside the context |
| `Text.tsx` | **Styled text primitive** — renders `<ink-text>` with color, bold/dim, italic, underline, strikethrough, inverse, and 8 wrap modes |
| `AlternateScreen.tsx` | **Alternate screen manager** — enters/exits DEC 1049, optionally enables SGR mouse tracking, constrains children to viewport height |

## How Files Relate to Each Other

```
App.tsx (root)
├── provides TerminalSizeContext → dimensions for all children
├── provides TerminalFocusContext → focus state for all children
├── provides AppContext (StdinContext) → stdin stream, raw mode, events
├── provides ClockProvider → animation timing
├── provides CursorDeclarationContext → native cursor for IME/accessibility
│
├── reads from TerminalSizeContext (own size)
├── reads from TerminalWriteContext (own write)
│
└── renders children wrapped in:
    └── AlternateScreen (when used by app)
        ├── reads TerminalSizeContext → height constraint
        ├── reads TerminalWriteContext → escape sequences
        ├── provides Box → flex container for children
        │
        └── children may use:
            ├── TerminalSizeContext → responsive layout
            ├── TerminalFocusContext → focus-aware UI
            ├── Text → styled text output
            └── AppContext → stdin interaction
```

**Key dependency chain:**

1. `App.tsx` is the **entry point** — it owns raw terminal mode, stdin/stdout streams, and event parsing. It reads terminal dimensions and writes escape sequences.

2. `TerminalSizeContext` and `TerminalFocusContext` are **extracted into separate files** from `App.tsx` specifically to prevent the root component from re-rendering when terminal state changes. Only context consumers re-render.

3. `AppContext` (StdinContext) is the **bridge between terminal I/O and React** — it exposes the stdin stream and raw mode control so that child components (e.g., interactive prompts) can manipulate terminal behavior.

4. `AlternateScreen` uses `TerminalSizeContext` and `TerminalWriteContext` to **manage the alternate screen buffer** (DEC 1049). It is a leaf-level orchestrator, not a leaf component — it wraps children and manages terminal state around them.

5. `Text` is a **leaf component** — it receives styling props and renders `<ink-text>`, relying on the reconciler to convert it into terminal output. It has no terminal I/O of its own.

## Key Takeaways

- **Context separation is a deliberate performance pattern.** `App.tsx` owns all terminal I/O but delegates state to focused contexts so that the root never re-renders from terminal events.

- **`useInsertionEffect` is critical** in `AlternateScreen.tsx` — it ensures escape sequences reach the terminal before React paints, preventing a visual flash on the main screen during mount/unmount.

- **`useSyncExternalStore` bridges external state** in `TerminalFocusContext.tsx`, connecting React's subscription model to the raw terminal focus events managed in `terminal-focus-state.js`.

- **Text wrapping is pre-memoized** in `Text.tsx` — eight distinct wrap modes are pre-configured flexbox objects, avoiding per-render style computation.

- **Raw mode is ref-counted** in `App.tsx` — multiple components may request raw mode, and it only disables when the last requestor releases it.

- **The architecture follows a clear layer order**: `App` (I/O) → Contexts (state) → Layout components (structure) → Leaf components (content).
