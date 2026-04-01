# Summary of `vim/`

## Purpose of `vim/`

The `vim/` directory implements a **pure Vim emulation layer** for a text editor. It provides complete support for Vim's modal editing paradigm, including:

- **NORMAL mode** keystroke interpretation
- **INSERT mode** character capture for dot-repeat
- Text **operators** (delete, change, yank, indent, paste, etc.)
- **Motions** (word movement, line navigation, find characters)
- **Text objects** (inner/around word, quotes, brackets)
- **Dot-repeat** (replay recorded changes via `.`)

The implementation is deliberately **side-effect free at the operator/motion level** — the state machine (`transitions.ts`) reads input and produces actions; the actions themselves are composed of pure functions that return new text/positions. Side effects (actual DOM/text updates) are delegated to the `TransitionContext` callbacks.

---

## Contents Overview

| File | Lines | Responsibility |
|------|-------|----------------|
| [`types.ts`](types.ts) | ~180 | **Type contracts** — all TypeScript interfaces, unions, discriminated types, and constants for the state machine |
| [`transitions.ts`](transitions.ts) | ~400 | **State machine** — input dispatch and state transitions for NORMAL mode |
| [`motions.ts`](motions.ts) | ~150 | **Motion resolution** — pure functions mapping motion keys (`h,j,k,l,w,b,e,0,$,G,gg`) to target `Cursor` positions |
| [`operators.ts`](operators.ts) | ~350 | **Operator execution** — pure functions implementing delete, change, yank, x, r, ~, J, p, P, indent, open-line |
| [`textObjects.ts`](textObjects.ts) | ~160 | **Text object finding** — pure functions locating boundaries of `iw`, `aw`, `i"`, `a(`, etc. |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                     Entry / Dispatch                        │
│  Keyboard input → TransitionContext                         │
│                      │                                       │
│                      ▼                                       │
│  transitions.ts ── transition()                            │
│                      │                                       │
│         ┌────────────┼────────────────────────────────┐    │
│         │            │                                │    │
│         ▼            ▼                                ▼    │
│  motions.ts    operators.ts                   textObjects.ts│
│  │           │                                        │    │
│  └───────────┼────────── Resolved positions ──────────┘    │
│              │ (Cursor offsets)                             │
│              ▼                                              │
│  ── Results returned to transitions ──►                     │
│                                              │              │
│                                              ▼              │
│  TransitionResult { next: VimState, execute?: Fn }         │
│                                              │              │
│                                              ▼              │
│                    Side effects via TransitionContext       │
│                    (setText, setOffset, recordChange, ...)  │
└─────────────────────────────────────────────────────────────┘
```

### Dependency graph (bottom-up)

```
types.ts        ← No dependencies (foundational types only)
       │
       ▼
motions.ts      ← imports Cursor (types)
       │
       ├──► textObjects.ts  ← imports Cursor, intl utils
       │
       ▼
operators.ts    ← imports Cursor, motions, textObjects, intl
       │
       ▼
transitions.ts  ← imports operators, motions, types
```

The bottom-up layering ensures each module depends only on stable abstractions below it:

1. **`types.ts`** defines the vocabulary (state shapes, constants, type guards)
2. **`motions.ts`** provides the primitive "cursor moves to X" operations
3. **`textObjects.ts`** uses motions concepts (character classes) to locate boundaries
4. **`operators.ts`** composes motions/text objects into higher-level edit operations
5. **`transitions.ts`** wires everything together as a state machine

---

## Key Takeaways

### 1. Separation of pure computation from side effects

Every module below `transitions.ts` is **pure**: given the same inputs (text, cursor, count), they always return the same output (new cursor, new text) with no observable side effects. Only `transitions.ts` calls the `TransitionContext` callbacks (`setText`, `setOffset`, `recordChange`). This makes the core logic trivially testable.

### 2. State machine drives everything

The 11-state `CommandState` machine in `types.ts` / `transitions.ts` is the central coordinator. Every keystroke in NORMAL mode is routed through it. The state machine pattern handles:

- **Count prefixes** (`3dw`, `50dd`)
- **Operator-pending mode** (waiting for a motion after `d`, `c`, `y`)
- **Chained commands** (`g`, `z`, `]` prefixes)
- **Dot-repeat recording** via `RecordedChange` discriminated union

### 3. Cursor is the universal currency

The `Cursor` type (from `../utils/Cursor.js`) is the universal interface between modules. Motions return cursors, operators take cursors as input, text objects locate cursor-relative boundaries. All position arithmetic is grapheme-aware, preventing corruption from multi-codepoint characters (emoji, combining marks, CJK).

### 4. Graceful fallback everywhere

Unknown keys, empty inputs, and edge cases never throw errors — they return `null`, the unchanged cursor, or reset to idle state. This ensures the editor never crashes on unexpected input sequences.

### 5. Dot-repeat is a first-class concept

`RecordedChange` in `types.ts` is an 11-variant discriminated union capturing the exact shape of every repeatable command. This enables accurate `.` replay for complex operations like `ci(`, `dG`, `>>`, `p`, not just simple motions.

### 6. Vim authenticity

The implementation respects Vim semantics that differ from naive interpretations:
- `cw` changes to end of current word, not start of next word
- `gj`/`gk` are characterwise exclusive (not linewise)
- `di(` excludes the brackets, `da(` includes surrounding whitespace
- `dd`/`yy` always yank with trailing newline
