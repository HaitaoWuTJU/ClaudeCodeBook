# Summary of `commands/keybindings/`

## Purpose of `keybindings/`

Provides a command-driven interface for users to customize keyboard shortcuts. Users can run a command to open or create their personal keybindings configuration file, which is then opened in their preferred editor for editing.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command registration entry point; defines metadata and lazy-loads the implementation |
| `keybindings.ts` | Command implementation; handles file creation, template generation, and editor invocation |
| `loadUserBindings.js` | Utilities for resolving the keybindings file path and checking feature availability |
| `template.js` | Generates default keybindings configuration content |
| `promptEditor.js` | Wrapper for opening files in the user's configured editor |
| `errors.js` | Shared error utility functions |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI invocation                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ index.ts (Command Registration)                                 │
│   • Validates feature flag via loadUserBindings.js             │
│   • Lazy-loads keybindings.ts via dynamic import()             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ keybindings.ts (Command Implementation)                         │
│   1. Checks feature flag (loadUserBindings.js)                  │
│   2. Gets keybindings file path (loadUserBindings.js)          │
│   3. Creates file with template (template.js)                   │
│   4. Opens editor (promptEditor.js)                             │
│   5. Returns structured result to CLI                           │
└─────────────────────────────────────────────────────────────────┘
```

**Lazy loading flow**: `index.ts` acts as the command registry entry point. When a user runs the keybindings command, the dynamic `import('./keybindings.js')` loads the actual implementation. This keeps the CLI startup fast by deferring I/O operations until needed.

**Data flow**: The command resolves the user's keybindings file path, generates a default template, creates the file atomically with exclusive flags, and opens it in the editor.

## Key Takeaways

- **Feature-gated command**: The entire keybindings feature is protected behind a preview feature flag. If disabled, the command is hidden from the CLI and returns a user-friendly preview message if called directly.
- **Atomic file creation**: Uses the exclusive `'wx'` file flag to create the keybindings file, preventing both TOCTOU race conditions and unnecessary `stat` calls. If the file already exists, it gracefully opens the existing file instead of failing.
- **Lazy loading architecture**: The command definition (`index.ts`) and implementation (`keybindings.ts`) are separated to optimize CLI startup time. Implementation is only loaded when the command executes.
- **Graceful error handling**: Returns structured `{ type: 'text'; value: string }` responses for all outcomes (created, already exists, editor unavailable), ensuring the CLI can present meaningful feedback to users.
- **No non-interactive support**: `supportsNonInteractive: false` reflects that this command requires human interaction (editing in an editor), so it cannot run in automated/CI environments.
