# Summary of `components/ClaudeCodeHint/`

## Purpose of `PluginHintMenu.jsx`

Displays a permission-style dialog prompting the user to install a plugin suggested by a command, with options to accept, decline, or decline and disable future plugin installation hints. Includes automatic dismissal after 30 seconds of inactivity.

## Contents Overview

| Element | Type | Description |
|---------|------|-------------|
| `AUTO_DISMISS_MS` | Constant | 30,000ms timeout value for auto-dismiss |
| `Props` | Interface | TypeScript type defining expected prop shape |
| `PluginHintMenu` | Function Component | Main exported component |
| `onResponseRef` | useRef | Stable reference to `onResponse` callback |
| `onSelect` | Handler | Processes user selection and triggers response |
| `options` | Array | Three selectable choices with labels and values |

## Data Flow

```
Props (pluginName, pluginDescription, marketplaceName, sourceCommand, onResponse)
        ↓
    useEffect (sets 30s timeout, updates onResponseRef.current)
        ↓
    Select Component (renders PermissionDialog with options)
        ↓
    User Interaction or Auto-dismiss (30s)
        ↓
    onSelect handler (maps selection to response)
        ↓
    onResponse('yes' | 'no' | 'disable')
```

## Key Implementation Details

- **Callback stability**: Uses `useRef` pattern (`onResponseRef.current = onResponse`) to ensure the timeout always calls the latest `onResponse` without re-triggering the effect
- **Auto-dismiss**: `useEffect` sets a 30-second timeout that calls `onResponse('no')` if no user interaction occurs; cleanup function clears timeout on unmount
- **Three-choice UI**: Options are rendered via `Select` inside a `PermissionDialog`:
  1. `"yes"` → `"Yes, install [pluginName]"`
  2. `"disable"` → `"No, and don't show plugin installation hints again"`
  3. `"no"` → `"No"` (default)
- **Conditional content**: `pluginDescription` only displays if provided (truthy check)
- **Cancel handling**: `Select` `onCancel` callback also triggers `onResponse('no')`

## Key Takeaways

- **Separation of concerns**: Dialog rendering (`PermissionDialog`, `Select`) is decoupled from business logic (plugin installation decision)
- **Timeout resilience**: The `useRef` pattern prevents stale closure issues where the timeout would call an outdated `onResponse` function
- **Cleaner user experience**: Provides a graceful "no response" fallback after 30 seconds rather than leaving the dialog hanging indefinitely
- **Preference persistence**: The `'disable'` option allows users to suppress future plugin installation prompts from the same source command
