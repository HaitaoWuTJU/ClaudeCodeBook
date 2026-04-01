# Summary of `context/`

## Purpose of `context/`

The `context/` directory houses React Context providers and custom hooks that manage **application-wide state, services, and rendering environment detection**. These contexts enable descendant components to access shared functionality without prop drilling, providing a clean separation of concerns between state logic and UI components.

---

## Contents Overview

The directory contains **7 context modules**, each addressing a distinct cross-cutting concern:

| File | Responsibility |
|------|----------------|
| `fps.tsx` | Distributes FPS (frames-per-second) performance metrics getter to interested components |
| `mailbox.tsx` | Provides a singleton `Mailbox` message/event utility via context |
| `modalContext.tsx` | Detects when components render inside a modal slot in `FullscreenLayout` and shares dimensions/scroll refs |
| `notifications.tsx` | Manages a prioritized notification queue with auto-dismiss, folding, and invalidation support |
| `useQueuedMessage.tsx` | Tracks queued message layout properties (padding, first-message status) |
| `stats.tsx` | Collects and aggregates session-level metrics (counters, gauges, histograms with reservoir sampling, sets) |
| `voice.tsx` | Manages voice feature state (recording, transcription, audio levels) with synchronous getters |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Shell                          │
│                         (App.jsx)                               │
└─────────────────────────────────────────────────────────────────┘
          │                  │                  │                │
          ▼                  ▼                  ▼                ▼
┌─────────────────┐ ┌──────────────┐ ┌──────────────────┐ ┌────────────┐
│  StatsProvider  │ │ VoiceProvider│ │NotificationsProv │ │ FpsMetrics │
│  (metrics flush │ │(voice state) │ │(queue manager)  │ │Provider    │
│   on exit)      │ │              │ │                  │ │            │
└────────┬────────┘ └──────┬───────┘ └────────┬─────────┘ └─────┬──────┘
         │                 │                  │                 │
         │                 ▼                  ▼                 │
         │        ┌──────────────┐  ┌──────────────────┐        │
         │        │  Mailbox     │  │QueuedMessage     │        │
         │        │  Provider    │  │Provider          │        │
         │        │  (singleton) │  │(padding/flags)   │        │
         │        └──────────────┘  └──────────────────┘        │
         │                                                        │
         ▼                                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Modal-aware components                          │
│              (FullscreenLayout sets ModalContext)                │
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────┐                   │
│  │ useIsInsideModal │◀───│ FullscreenLayout│                   │
│  │ useModalScrollRef│    │   (modal slot)   │                   │
│  │useModalOrTerminal│    └──────────────────┘                   │
│  └──────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

**Key relationships:**

1. **Layered providers**: `VoiceProvider`, `StatsProvider`, and `MailboxProvider` are typically mounted at the app root
2. **Modal detection**: `modalContext.tsx` is consumed by components inside `FullscreenLayout`'s modal slot to adapt their behavior (different sizing, suppressed dividers)
3. **Queued messages**: `useQueuedMessage.tsx` wraps message components that may appear in the modal or outside it, tracking layout differences
4. **Notifications flow**: The notification queue may trigger modal display (e.g., error notifications in a modal pane)
5. **Stats integration**: `stats.tsx` persists metrics via `saveCurrentProjectConfig`, which is also used by the broader application config system
6. **Mailbox utility**: The `Mailbox` singleton is a general-purpose message bus that any feature can use for inter-component communication

---

## Key Takeaways

### 1. Consistent Architecture
All contexts follow a similar pattern:
- Create context with `createContext`
- Provide context via a `<XxxProvider>` component
- Export `<XxxContext.Provider>` wrapping children
- Export custom hooks (`useXxx()`) for ergonomic access
- Throw descriptive errors when hooks are called outside providers

### 2. React Compiler Optimization
Every file uses `react/compiler-runtime` (`_c`) with cache slot arrays for automatic memoization, eliminating the need for manual `useMemo`/`useCallback` in most cases.

### 3. Synchronous State Patterns
- **`voice.tsx`**: Deliberately uses synchronous setters so callers can read new state immediately after setting (critical for keybinding handlers)
- **`stats.tsx`**: Aggregates data via `getAll()` and flushes to disk on `process.exit`
- **`notifications.tsx`**: Manages its own timeout tracking at module scope for immediate notification interruption

### 4. Specialized Data Structures
| Context | Data Structure | Unique Feature |
|---------|----------------|----------------|
| `stats.tsx` | Reservoir sampling (Algorithm R) | Percentile estimation without storing full dataset |
| `notifications.tsx` | Priority queue | Fold mechanism (like `Array.reduce()`) for deduplication |
| `modalContext.tsx` | Ref-based scroll control | `ScrollBoxHandle` for programmatic scroll reset |

### 5. Context as Dependency Injection
The contexts serve as **dependency injection containers**, decoupling feature logic from its consumers:
- `Mailbox` instance is created once and shared
- `StatsStore` aggregates metrics from throughout the app
- `VoiceStore` provides a unified voice feature API

### 6. Modal Slot Pattern
The `modalContext.tsx` establishes a **rendering environment detection** pattern where components can query their parent context rather than receiving props, enabling pluggable UI behavior based on where they're mounted.
