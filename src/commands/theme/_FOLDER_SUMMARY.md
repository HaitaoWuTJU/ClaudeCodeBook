# Summary of `commands/theme/`

## Purpose of `theme/`

This directory implements a CLI command that allows users to select and change the application's color theme. It follows a lazy-loading pattern where the command descriptor is separated from the actual implementation.

## Contents Overview

| File | Purpose |
|------|---------|
| `theme.ts` | Command descriptor—exports metadata (name, description, type) and a lazy-loading function for the actual implementation |
| `theme.tsx` | Command implementation—renders a themed picker UI using Ink/React, handles theme selection and cancellation |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────┐
│                         theme.ts                                 │
│  • Serves as the entry point for the command registry            │
│  • Exports metadata: name="theme", type="local-jsx"              │
│  • Provides lazy-load function: load: () => import('./theme.js') │
└──────────────────────────────┬───────────────────────────────────┘
                               │ (loaded on demand)
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                         theme.tsx                                │
│  • Contains ThemePickerCommand component                         │
│  • Wraps ThemePicker in a styled Pane                           │
│  • Uses useTheme() hook to update active theme                   │
│  • Exports call function implementing LocalJSXCommandCall        │
└──────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

- **Lazy Loading**: The actual theme implementation (`theme.tsx`) is not loaded until the user invokes the theme command, improving application startup performance.
- **Ink/React Based**: Both files use Ink (React for CLI) with TypeScript, enabling component-based CLI UI development.
- **Type-Safe Command Pattern**: Uses the `satisfies` operator and explicit `Command`/`LocalJSXCommandCall` types for compile-time safety.
- **Dynamic UI**: The `ThemePicker` component allows interactive selection from available themes, with immediate feedback via `useTheme()` hook.
- **Standardized Architecture**: Consistent with other commands in the codebase—separating descriptor metadata from implementation enables a centralized command registration system.
