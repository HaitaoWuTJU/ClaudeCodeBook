# Summary of `components/DesktopUpsell/`

# Desktop Upsell Module

## Purpose of `DesktopUpsell/`

The `DesktopUpsell/` directory contains functionality for promoting the desktop version of Claude Code to users who are currently using the CLI version. This is a **growth and user acquisition feature** designed to encourage adoption of the desktop application, which likely offers enhanced features like local model support and better system integration.

The upsell appears at startup, giving users a low-friction way to transition to the desktop experience.

## Contents Overview

The module implements a single, self-contained feature:

| File | Role |
|------|------|
| `index.tsx` | The entire upsell logic—configuration retrieval, platform detection, display logic, and dialog rendering |

The file is surprisingly comprehensive for a single module, handling:

- **Feature flag management** — Reads from GrowthBook dynamic config to enable/disable specific upsell behaviors (e.g., `enable_shortcut_tip`, `enable_startup_dialog`)
- **Platform detection** — Checks for macOS (`darwin`) or Windows x64, ensuring the desktop app is actually available for the user's OS
- **Visibility control** — Tracks `desktopUpsellSeenCount` to show the prompt at most 3 times, then stops
- **User preference persistence** — Writes `desktopUpsellDismissed` to global config when user selects "never show again"
- **Analytics integration** — Logs `tengu_desktop_upsell_shown` events with visibility count for funnel analysis
- **UI composition** — Renders a styled `PermissionDialog` containing a `Select` component for user choices

## How Files Relate to Each Other

Since this is a single-file module, the relationships are internal:

```
index.tsx
├── getDesktopUpsellConfig()
│   └── getDynamicConfig_CACHED_MAY_BE_STALE()     [external: GrowthBook SDK]
├── getGlobalConfig()                              [external: global state]
├── isSupportedPlatform()                          [internal: pure utility]
├── shouldShowDesktopUpsellStartup()
│   ├── isSupportedPlatform()                      [internal]
│   └── getDesktopUpsellConfig()                   [internal]
│   └── getGlobalConfig()                          [external: global state]
├── DesktopUpsellStartup (React component)
│   ├── useState (seen count, selection state)
│   ├── useEffect (increment count, log analytics)
│   │   └── saveGlobalConfig()                     [external: global state]
│   │   └── logEvent()                             [external: analytics]
│   └── render
│       ├── DesktopHandoff                         [external: handoff flow]
│       ├── PermissionDialog                       [external: UI primitives]
│       └── Select                                 [external: UI primitives]
└── _temp2 (updater for dismissal flag)
    └── saveGlobalConfig()                         [external: global state]
```

**External dependencies flow:**

```
GrowthBook SDK ──config──> getDesktopUpsellConfig()
    │                         └── enables/disables upsell features
    │
Global Config ──read──> getGlobalConfig()
    │                    └── checks dismissal, seen count
    │
    └──write──> saveGlobalConfig()
                    └── persists seen count, dismissal choice
    │
Analytics SDK ──write──> logEvent()
                      └── tracks upsell visibility

UI Primitives ──compose──> DesktopUpsellStartup
                      ├── PermissionDialog (wrapper)
                      ├── Select (user interaction)
                      └── DesktopHandoff (on "try" selection)
```

## Key Takeaways

1. **Platform-restricted** — The upsell only targets macOS and Windows x64 users; Linux and ARM Windows users are excluded because the desktop app isn't available for those platforms.

2. **Progressive opt-out** — Rather than a permanent "never show again" on first dismiss, users see the prompt up to 3 times before it's automatically suppressed. This balances user experience with acquisition goals.

3. **React 19 compiler compatibility** — The component uses `_c` symbol caching (React 19's compiler runtime pattern) and `react-compiler-runtime` imports, indicating it's designed to work with the React compiler for automatic memoization.

4. **Stateless read, stateful write** — Reads from a global config object (`getGlobalConfig`), but writes happen through React state updates triggered by effects, ensuring the component is reactive to config changes.

5. **No dead UI state** — If the user selects "not now" or cancels, the component gracefully unmounts without showing any fallback UI. The only two visible outcomes are the handoff screen ("try") or nothing at all.

6. **Analytics-first design** — Every mount logs an event with `seen_count`, making it straightforward to measure how many users see the prompt, convert to desktop, or explicitly dismiss.
