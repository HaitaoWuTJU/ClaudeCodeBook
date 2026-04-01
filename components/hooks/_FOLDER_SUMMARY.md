# Summary of `components/hooks/`

## Purpose of `components/hooks/`

The `hooks/` directory contains a suite of React components (built with Ink for CLI rendering) that implement a **read-only browser interface** for viewing configured hooks in a terminal application. These components allow users to navigate through hook events, matchers, and individual hook configurations without offering edit or delete functionality—the interface intentionally directs users to edit `settings.json` directly or ask Claude for modifications.

## Contents Overview

| File | Purpose |
|------|---------|
| `HooksConfigMenu.tsx` | Root container that manages navigation state machine across 4 modes |
| `PromptDialog.tsx` | Dialog for user responses to AI tool requests with selectable options |
| `SelectEventMode.tsx` | Entry point showing available hook events with hook counts |
| `SelectMatcherMode.tsx` | Displays matchers configured for a selected event |
| `SelectHookMode.tsx` | Lists all hooks for a selected event+matcher combination |
| `ViewHookMode.tsx` | Read-only detail view of a single hook configuration |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                     HooksConfigMenu                              │
│  (orchestrates modeState: select-event → select-matcher →        │
│   select-hook → view-hook)                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ renders based on modeState.mode
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│SelectEventMode│  │SelectMatcher  │  │ SelectHookMode│──┐
│ (initial view)│  │ Mode          │  │               │  │
└───────────────┘  └───────────────┘  └───────────────┘  │
                                                          ▼
                                                   ┌───────────────┐
                                                   │ ViewHookMode  │
                                                   │ (final state) │
                                                   └───────────────┘

Separate dialog chain:
┌───────────────┐
│ PromptDialog  │ ← Used independently for tool request responses
└───────────────┘
```

**Data flow** through the main menu:

1. `HooksConfigMenu` reads hook settings via `hooksConfigManager` and `getSettings_DEPRECATED()`
2. Computes `hooksByEvent`, `groupHooksByEventAndMatcher()`, `totalHooksCount`
3. Checks policy restrictions (`disableAllHooks`, `allowManagedHooksOnly`)
4. Renders `SelectEventMode` initially; based on user selection, transitions through modes
5. Each mode (`SelectMatcherMode` → `SelectHookMode` → `ViewHookMode`) receives filtered data as props
6. User interactions flow back via callbacks (`onSelectEvent`, `onSelectMatcher`, `onSelectHook`, `onCancel`)

## Key Takeaways

### Architecture
- **Mode-based navigation**: A state machine pattern where `HooksConfigMenu` holds `modeState` determining which sub-component renders
- **React Compiler**: All components use `_c` from `react/compiler-runtime` for aggressive memoization
- **Ink framework**: Terminal-native UI using `Box`, `Text`, `Select`, and `Dialog` components

### Read-Only Philosophy
- The entire menu is intentionally **read-only**—no add/edit/delete operations
- Users modifying hooks must edit `settings.json` directly or ask Claude
- Rationale: Supporting in-menu editing for all four hook types (command, prompt, agent, http) would create a maintenance burden

### Data Sources
- Hook configuration sourced from `hooksConfigManager` and `getSettingsForSource()`
- App state (tools, policies) accessed via `useAppState` / `useAppStateStore`
- Hooks organized by `groupHooksByEventAndMatcher()` utility

### Conditional Displays
- Policy restriction banners shown when hooks are globally disabled
- Matchers and plugins fields conditionally rendered based on availability
- Empty states with guidance when no hooks are configured
