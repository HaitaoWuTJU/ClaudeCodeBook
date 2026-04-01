# Summary of `ink/events/`

## Purpose of `events/`

The `events/` directory implements a complete DOM-like event system for Ink, a React renderer for CLI applications. It bridges low-level terminal I/O (keypresses, mouse clicks, focus changes) with high-level React component event handlers, providing familiar browser-like APIs (`onClick`, `onKeyDown`, `onFocus`) for terminal UI development.

## Contents Overview

| File | Role |
|------|------|
| `event.ts` | Base `Event` class providing `stopImmediatePropagation()` support |
| `terminal-event.ts` | Core `TerminalEvent` class implementing DOM-style propagation phases (capturing, at_target, bubbling) |
| `keyboard-event.ts` | Keyboard event with `key`, `ctrl`, `shift`, `meta`, `superKey`, `fn` properties |
| `input-event.ts` | Parses raw keypresses into structured `Key` objects and `input` strings via `parseKey()` |
| `click-event.ts` | Mouse click event with `col`, `row`, `localCol`, `localRow`, `cellIsBlank` |
| `focus-event.ts` | Focus/blur events with `relatedTarget` (uses focusin/focusout semantics) |
| `terminal-focus-event.ts` | Terminal-level focus events (DECSET 1004 protocol) |
| `event-handlers.ts` | Type definitions for all handler props and lookup tables (`HANDLER_FOR_EVENT`, `EVENT_HANDLER_PROPS`) |
| `dispatcher.ts` | Orchestrates two-phase event dispatch (capture → bubble) with React priority integration |
| `emitter.ts` | Custom `EventEmitter` extending Node's EventEmitter with propagation awareness |

## How Files Relate to Each Other

```
event.ts (Base Event with stopImmediatePropagation)
    │
    ├── terminal-event.ts (DOM propagation: bubbles, cancelable, phases)
    │       │
    │       ├── keyboard-event.ts (uses parseKey from input-event.ts)
    │       │
    │       ├── input-event.ts (parseKey function + InputEvent)
    │       │
    │       ├── click-event.ts (localCol/localRow recomputed per-target)
    │       │
    │       ├── focus-event.ts (relatedTarget, bubbles: true)
    │       │
    │       └── terminal-focus-event.ts (DECSET 1004 focus reporting)
    │
    └── emitter.ts (wraps Node EventEmitter, checks didStopImmediatePropagation)
            │
            └── dispatcher.ts (collects listeners, calls handlers with prepared events)

event-handlers.ts (type exports + HANDLER_FOR_EVENT lookup table)
    │
    └── dispatcher.ts (uses HANDLER_FOR_EVENT to find handlers on nodes)
```

**Dispatch flow:**
1. Terminal I/O triggers an event (keypress, mouse click, focus change)
2. `dispatcher.ts` collects handlers via `HANDLER_FOR_EVENT` lookup
3. Listeners are traversed in DOM order with capture/bubble phases
4. Each handler receives a prepared `TerminalEvent` subclass with correct `currentTarget`
5. `emitter.ts` wraps the dispatch, respecting `stopImmediatePropagation()`

## Key Takeaways

1. **DOM parity**: Ink replicates browser event semantics (capture/bubble phases, propagation control, preventDefault) for familiar React patterns
2. **Priority integration**: The dispatcher maps event types to React's `DiscreteEventPriority`, `ContinuousEventPriority`, and `DefaultEventPriority` for appropriate update scheduling
3. **Coordinate localization**: `ClickEvent` recalculates `localCol`/`localRow` for each dispatch target, providing coordinates relative to the current handler's Box
4. **Terminal protocols**: Ink handles multiple keyboard protocols (Kitty CSI u, xterm modifyOtherKeys, DEC application keypad) and focus reporting (DECSET 1004)
5. **Propagation control**: Two layers of stopping — `stopImmediatePropagation()` prevents sibling handlers on the same node; `stopPropagation()` prevents ancestor handlers
6. **Circular dependency avoidance**: `Dispatcher.discreteUpdates` is injected after construction to prevent circular imports with the reconciler
