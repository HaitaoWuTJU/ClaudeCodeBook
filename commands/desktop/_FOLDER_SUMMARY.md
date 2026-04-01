# Summary of `commands/desktop/`

## Purpose of `desktop/`

Provides a command implementation that enables users to hand off their current CLI session to Claude Desktop. The directory contains two files that work together: a command configuration entry point and the actual UI component.

## Contents Overview

| File | Type | Purpose |
|------|------|---------|
| `index.ts` | Command Config | Entry point with platform detection, lazy-loading, and Command interface compliance |
| `index.tsx` | React Component | Renders `DesktopHandoff` UI and handles the `onDone` callback |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                      desktop/                            │
│                                                          │
│  ┌──────────────────┐         ┌──────────────────────┐  │
│  │   index.ts       │         │   index.tsx          │  │
│  │                  │         │                      │  │
│  │  - Platform check│         │  - call(onDone)      │  │
│  │  - Lazy loading  │  ─────► │  - <DesktopHandoff>  │  │
│  │  - Command meta  │         │  - Promise<Node>     │  │
│  └──────────────────┘         └──────────────────────┘  │
│           │                           │                  │
└───────────│───────────────────────────│──────────────────┘
            │                           │
            ▼                           ▼
    Loaded when invoked          Renders the UI
    via import('./desktop.js')   via DesktopHandoff
```

**Load sequence**:
1. `index.ts` is imported → returns `desktop` command config with `load()` function
2. When command executes, `load()` dynamically imports `./desktop.js`
3. The dynamic import resolves to `index.tsx`'s `call(onDone)` function
4. `call()` renders `<DesktopHandoff>` component and returns Promise

## Key Takeaways

- **Platform gating**: Command only works on macOS (`darwin`) or Windows x64; silently hidden on other platforms
- **Lazy loading**: Actual implementation loaded only on demand via dynamic `import()`
- **Async command pattern**: Uses `Promise<ReactNode>` return type, standard for interactive commands
- **Separation of concerns**: Configuration/registration logic (`index.ts`) separated from rendering (`index.tsx`)
- **Callback-driven completion**: Uses `onDone(result, options)` pattern typical of command system
