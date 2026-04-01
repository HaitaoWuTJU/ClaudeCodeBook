# Summary of `plugins/`

## Purpose of `plugins/`

The `plugins/` directory serves as the foundation for the built-in plugin system in the CLI. Its purpose is to initialize plugins that:

- Ship with the CLI by default
- Appear in the `/plugin` UI for users to enable or disable
- Are distinct from bundled skills that have complex setup logic or auto-enabling behavior

This directory is currently **scaffolding** — it provides the structural foundation for a future migration path where certain bundled features will become user-toggleable plugins.

## Contents Overview

```
plugins/bundled/
└── index.ts          # Plugin initialization entry point (scaffolding)
```

| File | Purpose |
|------|---------|
| `index.ts` | Exports `initBuiltinPlugins()` function that will register built-in plugins at CLI startup |

## How Files Relate to Each Other

The `bundled/` directory is part of a larger plugin ecosystem:

```
src/
├── builtinPlugins.js          # Provides registerBuiltinPlugin() utility
├── skills/bundled/            # Contains skills with complex auto-enabling logic
└── plugins/                   # User-installed plugins

plugins/bundled/               # User-toggleable built-in plugins (this directory)
```

**Key distinction:**

- **`plugins/bundled/`** → User-controllable via `/plugin` UI
- **`src/skills/bundled/`** → Skills with complex setup that auto-enable (e.g., `claude-in-chrome`)

## Key Takeaways

1. **Scaffolding state**: The directory is currently empty of actual plugin implementations; `index.ts` contains only a placeholder function

2. **Plugin registration pattern**: Future plugins will be registered using `registerBuiltinPlugin()` imported from `../builtinPlugins.js`

3. **CLI startup integration**: `initBuiltinPlugins()` is intended to be called during CLI initialization to register available plugins

4. **Migration target**: This system appears designed to migrate features from `src/skills/bundled/` into a more accessible user-facing plugin model

5. **No external dependencies**: The current implementation has no npm dependencies, keeping the CLI bundle lightweight

6. **UI-ready architecture**: Plugins registered here will automatically appear in the `/plugin` UI, providing discoverability for users
