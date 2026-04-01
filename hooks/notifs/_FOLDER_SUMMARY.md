# Summary of `hooks/notifs/`

## Purpose of `notifs/`

The `notifs/` directory contains React hooks that manage user-facing notifications across a wide range of application states and events. Every hook in this directory contributes to the notification system by reading application state, evaluating conditions, and either adding a new notification via `useNotifications` or removing an existing one. The notifications serve as the primary channel for communicating important status changes, warnings, and suggestions to the user.

## Contents Overview

| File | Trigger | Notification |
|------|---------|--------------|
| `useAutoModeUnavailableNotification.tsx` | Mode wraps from a higher privilege to `'default'` with auto mode unavailable and opted-in | Warning about why auto mode is disabled (settings, circuit-breaker, org-allowlist) |
| `useCanSwitchToExistingSubscription.tsx` | Startup (via `useStartupNotification`), up to 3 times per session | Suggestion to run `/login` to activate existing Pro/Max subscription on Console |
| `useDeprecationWarningNotification.tsx` | A deprecated model is currently selected | High-priority warning with the deprecation message |
| `useLimitsNotification.tsx` | Rate limits or billing overage detected | Two separate notifications: overage message (immediate) and rate limit warning (high) |
| `useSettingsErrors.tsx` | One or more settings fail validation | Warning listing issue count with link to `/doctor` |
| `useStartupNotification.ts` | Component mount (one-shot) | Generic utility hook — the passed `compute` function determines the notification |
| `useTeammateShutdownNotification.ts` | In-process teammate tasks start or complete | Low-priority batched messages ("3 agents spawned") |

## How Files Relate to Each Other

The files form a layered architecture around a shared set of dependencies:

```
src/context/notifications.js
        │
        ├── useNotifications() ─── adds/removes notifications
        │                              │
        │   ┌──────────────────────────┴───────────────────────┐
        │   │                                                    │
        │ useAutoModeUnavailableNotification                    useDeprecationWarningNotification
        │ useCanSwitchToExistingSubscription                    useLimitsNotification
        │ useSettingsErrors                                     useTeammateShutdownNotification
        │ useStartupNotification                                useAutoModeUnavailableNotification
        │                                                      ...
        │
        └── Notification type (shared interface)
                 │
                 ├── key: string (deduplication identity)
                 ├── text / jsx: content
                 ├── color: "warning" | "error" | "suggestion"
                 └── priority: "low" | "high" | "immediate"
```

**`useStartupNotification.ts`** sits at the bottom as a reusable foundation — it abstracts the one-shot-on-mount pattern (remote mode guard + ref-based deduplication) that several other hooks reimplement manually. Other hooks that need a one-shot startup notification can delegate to it rather than duplicating the pattern.

All hooks share these common integration points:

| Integration | Shared Across |
|-------------|---------------|
| `getIsRemoteMode()` | All hooks check this before taking action; in remote mode, notifications are skipped entirely |
| `useNotifications` context | Every hook calls `addNotification` (or `removeNotification`) through this context |
| `useRef` deduplication | `useDeprecationWarningNotification`, `useTeammateShutdownNotification`, and the startup hook all use refs to prevent duplicate or repeated notifications |
| `useAppState` / `useSettingsChange` | State hooks feed reactive data into the notification logic, triggering re-evaluation when relevant state changes |

## Key Takeaways

1. **One-shot patterns dominate.** Most hooks are designed to fire a notification at most once per session (using `useRef` guards), or within a bounded number of times (e.g., `useCanSwitchToExistingSubscription` caps at 3). This prevents notification fatigue for persistent conditions.

2. **Remote mode is a universal kill switch.** Every hook calls `getIsRemoteMode()` and exits early if true. The notification system is treated as a local-mode-only feature.

3. **Notifications are typed and prioritized.** The `Notification` interface distinguishes between `priority` levels (`low`, `high`, `immediate`) and `color` variants (`warning`, `error`, `suggestion`). This allows the notification renderer to handle urgent alerts differently from informational ones.

4. **Fold-based batching is used for teammate events.** `useTeammateShutdownNotification` uses the `fold` property on notifications to combine rapid repeated events into a single count ("3 agents spawned"), reducing noise when many teammates operate concurrently.

5. **Settings and limits are reactive.** Unlike one-shot notifications, `useSettingsErrors` and `useLimitsNotification` remain live — they re-evaluate whenever their underlying state changes and add or remove their notification accordingly.

6. **The hook-based approach keeps logic colocated with effects.** Each notification concern is encapsulated in its own hook, making the notification system easy to extend (add a new hook) and easy to reason about (each hook has a single responsibility).
