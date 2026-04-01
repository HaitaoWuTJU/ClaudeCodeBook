# Summary of `ink/termio/`

## Purpose of `ink/termio/`

`termio/` is a **terminal I/O abstraction layer** for parsing and generating ANSI/ECMA-48 escape sequences. It converts raw terminal byte streams into semantic `Action` objects (cursor moves, style changes, mouse events) and provides inverse helpers for generating escape sequences from structured data.

## Contents Overview

| File | Role |
|------|------|
| `types.ts` | Semantic type definitions — `Action`, `TextStyle`, `Color`, `CursorAction`, etc. |
| `ansi.ts` | Low-level ANSI constants — C0 control characters, escape character, CSI/OSC/DCS/APC markers, byte-range predicates |
| `csi.ts` | CSI (Control Sequence Introducer) generator and type guards — cursor movement, erasure, scrolling, special markers |
| `dec.ts` | DEC Private Mode helpers — `decset()`/`decreset()` wrappers for modes 1–1049 (cursor, mouse, alt screen, etc.) |
| `osc.ts` | OSC (Operating System Command) generator and parser — window titles, hyperlinks, tab status |
| `sgr.ts` | SGR (Select Graphic Rendition) parser — decodes parameter strings into `TextStyle` objects |
| `esc.ts` | Simple ESC sequence parser — handles 2-character sequences (RIS, DECSC, DECRC, IND, RI, NEL) |
| `tokenize.ts` | Streaming input tokenizer — splits raw terminal input into text chunks and raw sequence strings |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────┐
│  types.ts                                                    │
│  (Semantic types: Action, TextStyle, Color, CursorAction...)  │
└──────────────────────────┬───────────────────────────────────┘
                           │ (defines types consumed by all)
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  ansi.ts                                                    │
│  (Constants: C0, ESC, SEP, ESC_TYPE; predicates)            │
└──────────┬─────────────────────────────────┬────────────────┘
           │                                 │
           ▼                                 ▼
┌─────────────────────┐           ┌─────────────────────────┐
│  csi.ts             │           │  osc.ts                 │
│  (CSI generators;   │           │  (OSC generators &     │
│   uses ESC from     │           │   parsers; uses ESC)    │
│   ansi.ts)          │           └──────────┬──────────────┘
└──────────┬──────────┘                        │
           │                                   ▼
           ▼                       ┌─────────────────────────┐
┌─────────────────────┐           │  sgr.ts                  │
│  dec.ts             │           │  (SGR parser; uses       │
│  (DEC mode helpers; │           │   TextStyle from types)  │
│   calls csi())      │           └──────────┬──────────────┘
└─────────────────────┘                        │
                                               ▼
┌─────────────────────────────────────────────────────────────┐
│  tokenize.ts                                                 │
│  (Main streaming tokenizer; orchestrates parsing using:     │
│   - isCSI* / isEscFinal from ansi.ts + csi.ts               │
│   - esc.ts (parseEsc) for simple ESC sequences              │
│   - osc.ts (parseOSC) for OSC 8/21337                       │
│   - sgr.ts (applySGR) for style changes)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
            Produces Action objects (from types.ts)
```

**Dependency chain summary:**

```
types.ts ────────────────────────────────► all files
                                               ▲
ansi.ts ──► csi.ts ──► dec.ts                  │
     │         │                               │
     │         └──────────────► osc.ts ◄───────┘
     │                              │
     └────────► sgr.ts ◄────────────┘
              (partial)

ansi.ts, csi.ts, osc.ts, sgr.ts ──► esc.ts (unused import)
                                    tokenize.ts (uses all)
```

## Key Takeaways

1. **Layered architecture** — Three tiers: constants (`ansi.ts`) → generators/parsers (`csi.ts`, `dec.ts`, `osc.ts`, `sgr.ts`, `esc.ts`) → orchestrator (`tokenize.ts`), all unified by semantic types in `types.ts`.

2. **Bidirectional encoding/decoding** — The library can both **generate** escape sequences (via `csi()`, `decset()`, `osc()` helpers) and **parse** them into semantic actions (via `tokenize()`, `parseEsc()`, `parseOSC()`, `applySGR()`).

3. **Streaming tokenizer** — `tokenize.ts` implements a state machine with 8 states (`ground`, `escape`, `csi`, `osc`, etc.) for incremental/streaming input processing, buffering incomplete sequences between `feed()` calls.

4. **Standards compliance** — Implements ECMA-48, ANSI X3.64, DEC private modes, xterm sequences, kitty keyboard protocol, and OSC 8 (hyperlinks) and OSC 21337 (per-tab metadata).

5. **No external dependencies** — The entire directory is self-contained, importing only from `./ansi.js` and `./types.js`.

6. **Semantic over raw** — Raw sequence strings are only exposed at the tokenizer boundary; internally the library operates on discriminated union `Action` objects for type-safe, exhaustive pattern-matching.
