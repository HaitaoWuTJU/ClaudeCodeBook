# Summary of `ink/hooks/`

## Purpose of `ink/hooks/`

Provides **React hooks** for Ink — a library for building CLIs with React. Each hook exposes a specific terminal capability to React components in a declarative, composable manner.

---

## Contents Overview

The hooks fall into three functional categories:

| Category | Hooks | Description |
|----------|-------|-------------|
| **Input / Interaction** | `useInput`, `useStdin` | Capture keyboard input and access stdin stream |
| **Terminal Metadata** | `useTerminalFocus`, `useTerminalTitle`, `useTabStatus` | Read/write terminal state: focus, window title, tab indicators |
| **Layout / Rendering** | `useAnimationFrame`, `useDeclaredCursor`, `useTerminalViewport` | Control animations, cursor position, and viewport visibility |
| **Application** | `useApp` | Access global app context (e.g., exit/unmount) |

---

## How Files Relate to Each Other

```
Terminal Infrastructure (shared by multiple hooks)
├── useStdin ────────────────────────────────┐
│   └── Provides: stdin stream, raw mode     │◄── useInput reads from here
├── TerminalSizeContext ─────────────────────┤
│   └── Provides: terminal dimensions       │◄── useTerminalViewport reads
└── TerminalWriteContext ────────────────────┤
    └── Provides: raw stdout writer         │◄── useTabStatus, useTerminalTitle write
                                               via osc() sequences

Cursor & Animation Coordination
├── CursorDeclarationContext ────────────────┐
│   └── Shared cursor parking state          │◄── useDeclaredCursor reads/writes
└── ClockContext ────────────────────────────┤
    └── Shared animation clock               │◄── useAnimationFrame reads
        (with keepAlive subscriber)

Visibility Bridge
└── useTerminalViewport ─────────────────────┐
    └── Computes: is element visible?         │◄── useAnimationFrame reads this
                                                to auto-pause offscreen
```

### Dependency Chain Example (`useInput`)

```
useInput
 ├── reads: useStdin (stdin stream + event emitter)
 ├── reads: useEventCallback (stable listener ref)
 └── writes: terminal raw mode (via setRawMode)
```

### Dependency Chain Example (`useAnimationFrame`)

```
useAnimationFrame
 ├── reads: ClockContext (shared time)
 ├── reads: useTerminalViewport (isVisible)
 └── writes: React state (setTime) on interval
```

---

## Key Takeaways

1. **Context-driven architecture**: Most hooks consume or produce context values (`StdinContext`, `TerminalWriteContext`, `ClockContext`, `CursorDeclarationContext`, etc.), enabling cross-cutting concerns without prop drilling.

2. **Terminal protocol awareness**: Several hooks emit raw escape sequences directly:
   - `useDeclaredCursor` — DECSET/DECRST for cursor visibility
   - `useTerminalTitle` — **OSC 0** for window/tab title
   - `useTabStatus` — **OSC 21337** for tab status indicators
   - `useTerminalFocus` — **DECSET 1004** for focus reporting

3. **Automatic optimizations**: Hooks avoid unnecessary work:
   - `useAnimationFrame` pauses when element is offscreen
   - `useTerminalViewport` updates refs without re-renders
   - `useInput` uses `useLayoutEffect` for synchronous raw mode setup

4. **Platform differentiation**: `useTerminalTitle` falls back to `process.title` on Windows (no OSC support), while Unix/macOS use full escape sequences.

5. **IME/CJK support**: `useDeclaredCursor` ensures input methods render preedit text inline at the correct cursor position, critical for CJK language input.

6. **Stable callback references**: `useEventCallback` (from `usehooks-ts`) wraps handlers to prevent re-subscriptions when closure values change.
