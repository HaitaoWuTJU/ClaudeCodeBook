# Summary of `components/LspRecommendation/`

## Purpose of `LspRecommendation/`

This directory contains a terminal UI component (`LspRecommendationMenu.jsx`) that displays a dialog prompting users to install LSP (Language Server Protocol) plugins for improved code intelligence features. It serves as part of a recommendation system that suggests language servers when users open files with specific extensions.

## Contents Overview

| File | Description |
|------|-------------|
| `LspRecommendationMenu.jsx` | Single component that renders an interactive dialog with four response options and a 30-second auto-dismiss timer |

The component presents users with four choices:
1. **Yes** — Install the recommended LSP plugin
2. **No** — Decline this time (also triggered by timeout)
3. **Never** — Skip this plugin forever
4. **Disable All** — Turn off all LSP recommendations globally

## How Files Relate to Each Other

```
LspRecommendationMenu.jsx
├── imports: ink.js (Box, Text)         → Terminal UI primitives
├── imports: CustomSelect/select.js     → Dropdown selection component
└── imports: permissions/PermissionDialog.js → Dialog wrapper/container
```

The component is self-contained — all logic for handling user responses and the auto-dismiss timer is encapsulated within `LspRecommendationMenu.jsx`. It expects to receive plugin information via props and communicates results back to the parent via the `onResponse` callback.

## Key Takeaways

- **Stable callback pattern**: Uses `useRef` to store the latest `onResponse` callback, preventing the auto-dismiss timer from resetting when parent components re-render with new callback references

- **Auto-dismiss timeout**: After 30 seconds of inactivity, the dialog automatically responds with `'no'`, treating the recommendation as ignored

- **Response mapping**: The `onSelect` handler uses a switch statement to map select component values (`'yes'`, `'no'`, `'never'`, `'disable'`) to the appropriate callback invocations

- **Terminal-compatible UI**: Built with Ink components (`Box`, `Text`) designed for CLI rendering, supporting terminal styling like dim colors and bold text

- **Conditional content**: Plugin description is only rendered if provided, making the dialog flexible for plugins with or without additional context
