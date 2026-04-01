# Summary of `commands/stats/`

## Purpose of `stats/`

The `stats/` directory defines the **"stats" command** for a CLI application, following a modular command architecture with lazy-loading and clear separation between configuration and implementation.

## Contents Overview

| File | Role |
|------|------|
| `stats.js` | Command configuration — metadata (name, type, description) and lazy-load entry point |
| `index.ts` | Internal barrel re-export of `stats.js` |
| `stats.tsx` | Command implementation — exports the `call` function that renders `<Stats>` UI |
| `package.json` | Package manifest for the command module |

## How Files Relate to Each Other

```
stats.js (exports command config)
    │
    │ load() function
    ▼
stats.tsx (exports call function)
    │
    ▼
<Stats /> component rendered
```

1. **`stats.js`** serves as the command registry entry point. It exports a configuration object with:
   - `type: 'local-jsx'` — identifies this as a local JSX-rendered command
   - `name: 'stats'` — the command identifier
   - `description` — user-facing help text
   - `load()` — a lazy import function that dynamically loads `./stats.jsx` only when the command is invoked

2. **`stats.tsx`** is the actual command implementation, loaded on-demand. It exports a `call` function conforming to `LocalJSXCommandCall`, which renders the `<Stats>` component and passes it an `onDone` callback for cleanup/closing.

3. **`index.ts`** provides an internal barrel export, likely used by other parts of the application to import the command without knowing the internal file structure.

## Key Takeaways

- **Lazy loading pattern**: The command implementation (`stats.tsx`) is not loaded until the user runs `stats`, improving CLI startup performance
- **Separation of concerns**: Command metadata lives in `stats.js`, while actual rendering logic lives in `stats.tsx`
- **TypeScript-native**: Uses `.tsx` extension for JSX support with full type safety via `LocalJSXCommandCall`
- **Callback-driven**: The `call` function receives an `onDone` callback, allowing the Stats UI to trigger parent-level actions (like closing a modal or exiting)
