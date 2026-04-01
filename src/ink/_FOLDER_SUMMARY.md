# Summary of `ink/`

# Ink Library Overview

**Ink** is a React renderer for building interactive CLI applications. It bridges React's component model with terminal I/O, enabling developers to write terminal UIs using familiar React patterns (hooks, context, components) with full support for styling, layout, input handling, mouse interactions, and ANSI escape sequences.

## Directory Structure

```
ink/
├── components/       # React UI components (App, Text, contexts)
├── events/           # DOM-like event system (keyboard, mouse, focus)
├── hooks/            # React hooks (useInput, useAnimationFrame, etc.)
├── layout/           # Yoga-based flexbox layout engine
├── termio/           # ANSI/ECMA-48 escape sequence parsing & generation
├── AlternateScreen.tsx
├── Ansi.tsx
├── bidi.ts
├── clearTerminal.ts
├── colorize.ts
├── config.ts
├── dom.tsx
├── line-width-cache.ts
├── measureElement.ts
├── new.tsx
├── offset-index.ts
├── patch-console.ts
├── reconciler.ts
├── render.tsx
├── Static.tsx
├── styles.ts
├── terminal.ts
├── terminalContext.tsx
├── useFocus.ts
├── useInput.ts
├── utils.ts
└── version.ts
```

## Top-Level File Summary

| File | Purpose |
|------|---------|
| `App.tsx` | Root React component — manages raw terminal I/O, stdin/stdout, keyboard/mouse parsing, error boundaries, and all major contexts |
| `render.tsx` | Entry point — creates a React root and renders into the terminal |
| `reconciler.ts` | Custom React reconciler — mounts Ink components to terminal output |
| `Static.tsx` | Component for non-React-managed output (bypasses reconciler) |
| `terminal.ts` | Terminal abstraction — escape sequence generators, progress reporting, title, clear screen |
| `dom.tsx` | DOM-like string builder — `ink-text`, `ink-box`, `ink-link` virtual DOM nodes |
| `styles.ts` | Shared style type definitions and named color mappings |
| `terminalContext.tsx` | `TerminalWriteContext` — raw write function for escape sequences |
| `useFocus.ts` | Hook returning `{ focused }` ref tracking focus within Ink components |
| `useInput.ts` | Hook for capturing terminal keyboard input |
| `config.ts` | Global Ink configuration (debug mode, experimental features) |
| `Ansi.tsx` | Parses and renders ANSI-colored strings |
| `bidi.ts` | Bidirectional text reordering for RTL scripts |
| `clearTerminal.ts` | Cross-platform terminal clearing with scrollback support |
| `colorize.ts` | Chalk-based colorization with tmux/VSCode compatibility |
| `line-width-cache.ts` | Memoized line width calculation with LRU cache |
| `measureElement.ts` | Measure rendered Box dimensions via resize observer |
| `new.tsx` | `New` component for dynamic content injection |
| `offset-index.ts` | Maps DOM-like string indices to ANSI-aware positions |
| `patch-console.ts` | Patches `console.log` to render styled output in Ink |
| `version.ts` | Exports `VERSION` string |

## How Everything Connects

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Developer Code                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ import {render, Text, Box, useInput} from 'ink'                       │ │
│  │                                                                       │ │
│  │ const MyApp = () => <Box><Text bold>Hello</Text></Box>               │ │
│  │                                                                       │ │
│  │ render(<MyApp />)                                                    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  render.tsx                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ 1. Creates React root via reconciler                                   │ │
│  │ 2. Mounts App (<ink-app>) component                                     │ │
│  │ 3. Provides TerminalWriteContext                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  reconciler.ts          App.tsx            AppContext.ts                    │
│  ┌─────────────┐        ┌─────────┐       ┌─────────────────────────┐      │
│  │ React       │───────►│ Root    │───────►│ StdinContext            │      │
│  │ Reconciler  │        │ Component│        │ (stdin, raw mode,      │      │
│  │             │        │         │        │  event emitter)        │      │
│  │ Mounts:     │        │ Provides:│        └─────────────────────────┘      │
│  │ • Text      │        │ • TerminalSizeContext │                         │
│  │ • Box       │        │ • TerminalFocusContext │                         │
│  │ • Link      │        │ • TerminalWriteContext  │                         │
│  │ • Static    │        │ • CursorDeclarationContext                       │
│  │             │        │ • ClockContext │                                │
│  │ Converts:   │        └──────────────┘                                 │
│  │ React tree  │                                                              │
│  │     ↓       │                                                              │
│  │ dom.tsx     │                                                              │
│  │ (ink-text,  │                                                              │
│  │  ink-box)   │                                                              │
│  └─────────────┘                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  dom.tsx                components/Text.tsx        components/Box.tsx        │
│  ┌─────────────┐        ┌──────────────┐          ┌───────────────────┐      │
│  │ ink-text    │◄───────│ Text         │          │ Box               │      │
│  │ ink-box     │        │ • color      │          │ • flexDirection   │      │
│  │ ink-link    │        │ • bold/dim   │          │ • width/height    │      │
│  │             │        │ • italic     │          │ • margin/padding │      │
│  │ Virtual DOM │        │ • underline  │          │                   │      │
│  │ nodes that  │        │ • wrap modes │          │ Uses:             │      │
│  │ encode      │        │              │          │ • layout/node.ts  │      │
│  │ escape      │        │ Maps props   │─────────►│ • layout/yoga.ts  │      │
│  │ sequences   │        │ to escape    │          │                   │      │
│  │ and content │        │ sequences    │          │ Handles:          │      │
│  └─────────────┘        └──────────────┘          │ • Text children   │      │
│                                                    │ • Nested boxes    │      │
│                                                    └───────────────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Terminal I/O Layer                                                          │
│                                                                              │
│  ┌───────────────────┐  ┌─────────────────┐  ┌──────────────────────────┐   │
│  │ TerminalWriteCtx  │  │ StdinContext    │  │ termio/                  │   │
│  │ (raw write)       │  │ (raw mode,      │  │ (ANSI escape sequences)  │   │
│  │                   │  │  events)        │  │                          │   │
│  │ writeRaw(string)  │  │                 │  │ tokenize.ts → Actions   │   │
│  │                   │  │ useInput hook   │  │ csi.ts → sequences      │   │
│  │ Used by:          │  │                 │  │ osc.ts → titles, links  │   │
│  │ • Ansi.tsx        │  │ Parses:         │  │ sgr.ts → colors/styles │   │
│  │ • AlternateScreen │  │ • Keyboard      │  │ dec.ts → DEC modes      │   │
│  │ • clearTerminal   │  │ • Mouse (CSI)   │  │                          │   │
│  │ • hooks/          │  │ • Focus (DEC 1004)  │                          │   │
│  └───────────────────┘  └─────────────────┘  └──────────────────────────┘   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  hooks/                    events/           layout/                   │ │
│  │  ┌──────────────────┐     ┌────────────┐   ┌─────────────────────┐    │ │
│  │  │ useInput         │────►│ dispatcher  │   │ layout/             │    │ │
│  │  │ useTerminalFocus │     │             │   │ Uses flexbox (Yoga) │    │ │
│  │  │ useAnimationFrame│     │ keyboard-   │   │ to compute Box      │    │ │
│  │  │ useTabStatus     │     │ event.ts    │   │ dimensions.         │    │ │
│  │  │ useTerminalTitle │     │ click-event │   │                     │    │ │
│  │  │ useDeclaredCursor│     │ focus-event │   │ node.ts → interface │    │ │
│  │  └──────────────────┘     └────────────┘   │ yoga.ts → impl       │    │ │
│  └────────────────────────────────────┬────────┘   │ geometry.ts → utils│    │ │
│                                       │           └─────────────────────┘    │ │
│                                       ▼                                       │ │
│                                    App.tsx ◄──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Actual Terminal Output                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ ESC[3J           ← clear scrollback                                    │  │
│  │ ESC[?1049h       ← alternate screen                                    │  │
│  │ ESC[?1000h       ← mouse tracking                                      │  │
│  │ ESC[?1004h       ← focus reporting                                      │  │
│  │ ESC[0;0H         ← cursor home                                         │  │
│  │ ┌────────────┐                                                           │  │
│  │ │  Terminal  │  ← React-rendered UI                                     │  │
│  │ │  Output    │    (Box layout, styled Text, Links)                       │  │
│  │ └────────────┘                                                           │  │
│  │ ESC[?1004l       ← disable focus reporting                              │  │
│  │ ESC[?1000l       ← disable mouse tracking                               │  │
│  │ ESC[?1049l       ← restore main screen                                 │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

### 1. React ↔ Terminal Bridge
Ink is not a CLI library with React-like syntax — it is actual React running in a Node.js environment. The custom reconciler (`reconciler.ts`) handles the mounting, updating, and unmounting of React components, converting them into terminal escape sequences via `dom.tsx`.

### 2. Layered Architecture
The codebase is organized into clear layers:

| Layer | Location | Responsibility |
|-------|----------|----------------|
| **Developer API** | Top-level exports | What users import (`render`, `Text`, `Box`, hooks) |
| **Component Model** | `components/` | React components (`App`, `Text`, `Box`, contexts) |
| **Rendering** | `reconciler.ts`, `dom.tsx` | React → virtual DOM nodes → string output |
| **Styling** | `styles.ts` | Type definitions, named colors |
| **Layout** | `layout/` | Yoga flexbox engine wrapper |
| **Events** | `events/` | Keyboard, mouse, focus event system |
| **Terminal I/O** | `termio/` | ANSI escape sequence parsing/generation |
| **Hooks** | `hooks/` | Terminal capabilities exposed as React hooks |

### 3. Context-Based State Management
Rather than a single monolithic component, Ink uses React Context to separate concerns:
- `TerminalWriteContext` — raw terminal output
- `TerminalSizeContext` — terminal dimensions
- `TerminalFocusContext` — terminal focus tracking
- `StdinContext` (AppContext) — stdin stream and raw mode
- `ClockContext` — shared animation timing
- `CursorDeclarationContext` — cursor visibility for IME/CJK

This prevents the root `App` component from re-rendering when only a specific piece of state changes.

### 4. ANSI Protocol Coverage
The `termio/` directory implements extensive terminal protocol support:
- **CSI sequences** — cursor movement, erasure, scrolling, DEC modes
- **OSC sequences** — window titles (OSC 0), hyperlinks (OSC 8), tab status (OSC 21337)
- **SGR codes** — colors, bold, dim, italic, underline, strikethrough, inverse
- **DEC private modes** — alternate screen (1049), mouse tracking (1000/1006), focus reporting (1004)
- **xterm / Kitty keyboard** — Unicode input via CSI u

### 5. Cross-Platform Considerations
Multiple files handle platform differences:
- `bidi.ts` — enables RTL text reordering on Windows and VS Code (which lack native support)
- `clearTerminal.ts` — different clear sequences for modern vs. legacy Windows terminals
- `colorize.ts` — boosts chalk level for VS Code, clamps for tmux compatibility
- `Ansi.tsx` — named color mapping ensures consistency across terminals

### 6. Performance Optimizations
- **Memoization** — `Text` pre-creates 8 wrap-mode style objects; `Ansi` is memoized
- **LRU cache** — `line-width-cache.ts` caches width calculations to avoid repeated computation
- **Animation pausing** — `useAnimationFrame` stops ticking when the component is offscreen
- **Viewport refs** — `useTerminalViewport` updates refs without triggering re-renders
- **Span merging** — `Ansi.tsx` merges consecutive styled spans to reduce React nodes

### 7. Progressive Enhancement
Terminal capabilities are detected at runtime:
- `needsBidi()` checks platform and terminal emulator
- `isModernWindowsTerminal()` detects Windows Terminal, VS Code, mintty
- `isProgressReportingAvailable()` checks for ConPTY/iTerm2/ConEmu
- `hasRTLCharacters()` uses regex to skip unnecessary bidi processing

---

## Subdirectory Summaries

For detailed analysis of each subdirectory, see:

- **[`ink/components/` →](ink/components/)** — Core React components: `App.tsx`, `Text.tsx`, contexts, and `AlternateScreen`
- **[`ink/events/` →](ink/events/)** — DOM-like event system with capture/bubble phases for keyboard, mouse, and focus
- **[`ink/hooks/` →](ink/hooks/)** — React hooks exposing terminal capabilities
- **[`ink/layout/` →](ink/layout/)** — Yoga-based flexbox layout engine
- **[`ink/termio/` →](ink/termio/)** — ANSI escape sequence parsing and generation
