# Summary of `commands/model/`

## Purpose of `model/`

The `model/` directory contains the command implementation for the `/model` CLI command, which allows users to manage the AI model that Claude Code uses for its main loop and planning operations.

## Contents Overview

| File | Purpose |
|------|---------|
| `model.tsx` (or `model.ts`) | Command descriptor that defines metadata (name, description, type) and lazy-loads the actual implementation |

The command descriptor exports a configuration object satisfying the `Command` interface:

- **`name: 'model'`** — Invoked as `/model`
- **`type: 'local-jsx'`** — Indicates it's a local JSX-based command with React UI
- **`description` getter** — Dynamically shows the currently active model (e.g., `"Set the AI model for Claude Code (currently Sonnet 4)"`)
- **`argumentHint: '[model]'`** — Shows expected argument format to users
- **`immediate` getter** — Determines if the command should auto-execute based on inference configuration
- **`load`** — Dynamic import of `./model.js` for lazy code-splitting

## How Files Relate to Each Other

```
model/
└── model.tsx (command descriptor)
    │
    └── imports & lazy-loads → model.js (actual command implementation)
                                  │
                                  ├── ModelPicker.js (interactive UI component)
                                  ├── model/ (utilities)
                                  │   ├── modelAliases.js (known aliases)
                                  │   ├── modelDefaults.js (default model)
                                  │   ├── modelAllowlist.js (org restrictions)
                                  │   ├── validateModel.js (validation logic)
                                  │   ├── modelSupports.js (capability checks)
                                  │   └── getMainLoopModel.js (state getter)
                                  │
                                  ├── analytics/ (event tracking)
                                  │
                                  └── effort.js, fastMode.js, extraUsage.js (feature flags)
```

The command descriptor (`model.tsx`) acts as a **thin facade** that:
1. Reads current state (via `getMainLoopModel()`)
2. Determines execution mode (via `shouldInferenceConfigCommandBeImmediate()`)
3. Delegates all logic to the dynamically imported `model.js`

## Key Takeaways

1. **Lazy Loading Pattern** — The actual command code is split into a separate chunk and only loaded when the user runs `/model`, keeping the initial bundle small

2. **Dynamic Descriptions** — The command's description updates in real-time to reflect the current model, helping users know what they're changing from

3. **Immediate Execution Support** — The command can optionally run immediately (skipping confirmation) based on inference config, useful for scripting

4. **Three Execution Modes**:
   - **Interactive picker** (`/model`) — Opens UI with model options
   - **Inline display** (`/model -i`) — Shows current model info
   - **Direct set** (`/model claude-opus-4-5`) — Validates and sets model directly

5. **State Management** — Tracks both the persistent `mainLoopModel` and the session-scoped `mainLoopModelForSession` (plan mode override), showing users when a temporary override is active

6. **Feature Integration** — The command coordinates fast mode, extra usage billing, effort level, and 1M context access checks when changing models
