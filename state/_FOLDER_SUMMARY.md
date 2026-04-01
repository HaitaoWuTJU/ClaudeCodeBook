# Summary of `state/`

## Purpose of `state/`

The `state/` directory is the **central nervous system for application state management** in this React/React Native application. It provides a layered architecture combining:

- A minimal, framework-agnostic store implementation
- The central `AppState` type definitions with default initialization
- React context providers and hooks for component integration
- A single choke point for state change side effects (settings sync, analytics, permission mode propagation)
- Pure selector functions for derived state
- UI helpers for managing teammate transcript views

---

## Contents Overview

| File | Role |
|------|------|
| **`store.ts`** | Low-level, framework-agnostic state store with immutable updates, subscriptions, and optional `onChange` callback |
| **`AppStateStore.ts`** | `AppState` type definitions + `getDefaultAppState()` factory (all state shape, defaults, speculation state) |
| **`AppState.tsx`** | React integration: `AppStateProvider`, `useAppState`, `useSetAppState`, `useAppStateStore`, `useAppStateMaybeOutsideOfProvider` |
| **`onChangeAppState.ts`** | Centralized side-effect handler: settings persistence, CCR/SDK permission-mode sync, cache invalidation, environment variable propagation |
| **`selectors.ts`** | Pure selector functions: `getViewedTeammateTask`, `getActiveAgentForInput` |
| **`teammateViewHelpers.ts`** | UI helpers: `enterTeammateView`, `exitTeammateView`, `release`, `stopOrDismissAgent` |

---

## How Files Relate to Each Other

```
                              ┌──────────────────────────────┐
                              │      getDefaultAppState()    │
                              │    (AppStateStore.ts)        │
                              └──────────────┬───────────────┘
                                             │ produces AppState
                                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           createStore()                                  │
│                         (store.ts)                                      │
│                                                                         │
│   Returns Store<AppState>:                                              │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  { getState, setState, subscribe }  ← immutable updates       │     │
│   └───────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        AppStateProvider (AppState.tsx)                    │
│                                                                          │
│   ┌──────────────────────────────────────────────────────────────────┐   │
│   │  • Creates store via useState(initializer)                       │   │
│   │  • Subscribes store to onChangeAppState()                        │   │
│   │  • Wraps: <MailboxProvider> <VoiceProvider?> {children}          │   │
│   │  • useSettingsChange → applySettingsChange → setState            │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Exposes via context:                                                   │
│   ┌─────────────────┐ ┌──────────────────┐ ┌────────────────────────┐   │
│   │  useAppState    │ │  useSetAppState  │ │  useAppStateStore      │   │
│   │  (subscribe +   │ │  (stable updater │ │  (raw store access for │   │
│   │   selector)     │ │   — no render)   │ │   non-React code)      │   │
│   └─────────────────┘ └──────────────────┘ └────────────────────────┘   │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    Consumer Components / Utilities                       │
│                                                                          │
│  ┌────────────────────┐     ┌────────────────────┐     ┌─────────────┐  │
│  │   selectors.ts     │     │ teammateViewHelpers│     │ Any component│  │
│  │                    │     │                    │     │  calling     │  │
│  │ getViewedTeammate  │     │ enterTeammateView()│     │ useAppState()│  │
│  │ getActiveAgent     │     │ exitTeammateView() │     │              │  │
│  └────────────────────┘     │ stopOrDismiss()   │     └─────────────┘  │
│                             └────────────────────┘                       │
└──────────────────────────────────────────────────────────────────────────┘
```

### Dependency Chain (bottom-up)

| Layer | Files | Responsibility |
|-------|-------|----------------|
| **Foundation** | `store.ts` | Core store: `getState`, `setState`, `subscribe` |
| **Types & Defaults** | `AppStateStore.ts` | `AppState` interface, `getDefaultAppState()` |
| **React Bridge** | `AppState.tsx` | Context, hooks, provider, settings sync |
| **Side Effects** | `onChangeAppState.ts` | Subscribes to store; handles permissions, settings, caches |
| **Utilities** | `selectors.ts`, `teammateViewHelpers.ts` | Pure selectors + UI helpers consuming state |

---

## Key Takeaways

### 1. Two-Store Architecture
The app maintains both an **external settings store** (file-watched/remote) and an **internal app state store**. `AppStateProvider` bridges them via `useSettingsChange` → `applySettingsChange` → `setState`.

### 2. Single Choke Point for Side Effects
`onChangeAppState` is the **only** function that subscribes to the store's `onChange` callback. All state transitions (Shift+Tab, dialogs, slash commands, rewind, REPL bridge) funnel through it, ensuring CCR and SDK are always notified of permission-mode changes.

### 3. Immutable, Selector-Based React Integration
- `useAppState(selector)` uses `useSyncExternalStore` with `Object.is` comparison — components only re-render when the *selected slice* changes
- `useSetAppState` returns a stable reference — components using only setters never re-render

### 4. Speculation State for Performance
Speculation state uses `{ current: ... }` ref objects instead of direct values, avoiding array re-spreading on every message during streaming.

### 5. Cycle-Breaking Patterns
`teammateViewHelpers.ts` inlines `PANEL_GRACE_MS` and `isLocalAgent()` to break circular dependencies that would otherwise occur through `BackgroundTasksDialog`.

### 6. Ultraplan Gating
The `isUltraplanMode` field is set only on the `false → true` transition (RFC 7396: `null` otherwise), ensuring ultraplan mode persists for exactly one plan cycle.

### 7. Feature-Gated Components
- `VoiceProvider` is tree-shaken in external builds via `feature('VOICE_MODE')` require
- `tungstenPanelVisible` persistence is ant-only (`USER_TYPE === 'ant'`)
