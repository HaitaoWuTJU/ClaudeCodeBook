# Summary of `commands/sandbox-toggle/`

## Purpose of `sandbox-toggle/`

The `sandbox-toggle/` directory implements a CLI command for managing sandboxing features in a development tool (likely Claude Code). Sandbox mode restricts command execution for security purposes, and this directory provides both the command registration and implementation to query, configure, and manage sandbox settings.

## Contents Overview

| File | Role | Description |
|------|------|-------------|
| `sandbox-toggle.ts` | Command definition | Registers the `sandbox` command with Ink, defines the description, icon, and status text based on current sandbox state. |
| `sandbox-toggle.tsx` | Command implementation | Handles command execution, validates platform/policy constraints, renders the interactive settings UI, and processes the `exclude` subcommand. |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│  sandbox-toggle.ts (Command Definition)                     │
│  ├─ Exports `command` object (name, description, etc.)     │
│  ├─ Dynamically builds description with status text        │
│  ├─ Checks platform support via SandboxManager             │
│  └─ Lazy-loads actual component via load()                 │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼ (load() imports on demand)
┌─────────────────────────────────────────────────────────────┐
│  sandbox-toggle.tsx (Command Implementation)               │
│  ├─ Main `call()` function                                  │
│  ├─ Platform/policy validation                              │
│  ├─ Dependency checking                                    │
│  ├─ Renders <SandboxSettings> UI component                 │
│  └─ Handles "exclude <pattern>" subcommand                 │
└─────────────────────────────────────────────────────────────┘
```

**Data flow between files:**

1. `sandbox-toggle.ts` defines the CLI interface; `sandbox-toggle.tsx` provides the runtime logic
2. Both import from `SandboxManager` in `../../utils/sandbox/sandbox-adapter.js`
3. The `command` object in `.ts` references the `.tsx` file via `load()`

## Key Takeaways

- **Command routing**: The command name is `sandbox` (registered via `sandbox-toggle.ts`), while `sandbox-toggle.tsx` handles the actual execution logic with `call()`.

- **Multi-state status**: The command displays nuanced status based on five boolean conditions (enabled, auto-allow, fallback allowed, managed, dependencies present).

- **Platform gating**: Both files perform platform checks—`.ts` hides the command entirely on unsupported platforms; `.tsx` shows targeted errors for WSL1 users vs other unsupported platforms.

- **Enterprise support**: Undocumented `enabledPlatforms` setting and `areSandboxSettingsLockedByPolicy()` allow organizations to enforce sandbox configurations.

- **Interactive + command-line modes**: Users can view status by running `sandbox` alone (interactive UI), or use `sandbox exclude "<pattern>"` for quick CLI exclusion management.

- **Dependency awareness**: Both files check `SandboxManager.checkDependencies()` and display warning icons when sandbox prerequisites are missing.
