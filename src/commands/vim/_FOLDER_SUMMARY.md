# Summary of `commands/vim/`

## Purpose of `vim/`

Provides a complete **editor mode toggle command** that switches between Vim and Normal editing modes, with the actual implementation decoupled from its metadata via lazy-loading.

## Contents Overview

| File | Role | Type |
|------|------|------|
| `Command.ts` | Command configuration & entry point | Static config |
| `vim.ts` | Command implementation & logic | Dynamic/lazy-loaded |

## How Files Relate to Each Other

```
┌─────────────────────────────────┐
│  Command.ts (config)            │
│  ├─ name: 'vim'                 │
│  ├─ description: '...'          │
│  ├─ type: 'local'               │
│  ├─ supportsNonInteractive: false
│  └─ load() → import('./vim')    │───────┐
└─────────────────────────────────┘       │
                                          ▼
                          ┌───────────────────────────┐
                          │  vim.ts (implementation)  │
                          │  ├─ getGlobalConfig()      │
                          │  ├─ toggle editorMode      │
                          │  ├─ saveGlobalConfig()     │
                          │  ├─ logEvent(...)          │
                          │  └─ return { type, value } │
                          └───────────────────────────┘
```

**Relationship**: `Command.ts` acts as a **façade** that exports the command's metadata and a lazy-loading entry point. When the command is invoked, the `load()` function dynamically imports `vim.ts`, which contains the actual toggle logic. This separation allows:

- **Fast CLI startup** — implementation code is not loaded until needed
- **Clean separation** — metadata (description, flags, types) stays distinct from business logic

## Key Takeaways

1. **Lazy-loading pattern**: The `load()` function returns a Promise that resolves to the `vim.ts` module, deferring the actual code execution until the command is called.

2. **Local command type**: `type: 'local'` indicates this is a workspace/project-specific command, not a built-in CLI feature.

3. **Non-interactive incompatibility**: `supportsNonInteractive: false` means this command requires an interactive terminal (likely for displaying the mode-specific help text).

4. **Config persistence**: The command reads/writes the `editorMode` field in the global config, making the mode choice persistent across sessions.

5. **Analytics integration**: Every mode change triggers a `tengu_editor_mode_changed` analytics event with the new mode and source.
