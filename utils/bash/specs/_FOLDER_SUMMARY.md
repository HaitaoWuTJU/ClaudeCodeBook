# Summary of `utils/bash/specs/`

## Purpose of `specs/`

The `specs/` directory serves as the **command specification registry** for a bash utilities module. It contains declarative TypeScript files that define the CLI schemas for various shell commands (e.g., `nohup`, `pyright`, `sleep`, `timeout`), enabling a consistent, type-safe way to configure and consume command-line interfaces.

## Contents Overview

| File | Command | Description |
|------|---------|-------------|
| `index.ts` | — | Central aggregation point; exports all specs as a single array |
| `alias.ts` | `alias` | Define or list shell aliases (`name=value` format) |
| `nohup.ts` | `nohup` | Run commands immune to hangups |
| `pyright.ts` | `pyright` | Python type checker with 21 CLI options |
| `sleep.ts` | `sleep` | Delay execution for a specified duration |
| `srun.ts` | `srun` | Execute commands on SLURM cluster nodes |
| `time.ts` | `time` | Measure command execution time |
| `timeout.ts` | `timeout` | Run commands with a time limit |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────┐
│            registry.ts (parent)         │
│         (defines CommandSpec type)      │
└──────────────────┬──────────────────────┘
                   │ imports
                   ▼
┌─────────────────────────────────────────┐
│              specs/index.ts             │
│   (aggregates all command specifications)│
└──────┬───────┬───────┬───────┬──────────┘
       │       │       │       │
       ▼       ▼       ▼       ▼
   ┌────┐  ┌──────┐  ┌────┐  ┌───────┐
   │alias│  │nohup │  │sleep│  │timeout│
   └────┘  └──────┘  └────┘  └───────┘
                  ┌──────┐  ┌────┐  ┌────┐
                  │srun  │  │time│  │pyright│
                  └──────┘  └────┘  └────┘
```

1. **`registry.ts`** defines the `CommandSpec` interface (imported by all spec files)
2. **`index.ts`** imports each individual spec and re-exports them as a consolidated array
3. **Individual specs** (`alias.ts`, `nohup.ts`, etc.) each export a single `CommandSpec` object
4. **Consumers** (CLI runners, help generators) import from `index.ts` to access all command schemas

## Key Takeaways

- **Pure Configuration**: All files in `specs/` are declarative—they define metadata, not runtime logic
- **Typed Schema Pattern**: Uses TypeScript's `CommandSpec` type (and `satisfies`) to validate specs at compile time
- **Variadic & Optional Arguments**: Commands like `alias` and `pyright` support flexible argument handling (`isVariadic`, `isOptional`)
- **Command vs. Value**: The `isCommand: true` flag distinguishes executable commands from plain arguments (used in `nohup`, `srun`, `time`, `timeout`)
- **Extensibility**: Adding a new command requires only creating a new `.ts` file and importing it in `index.ts`
- **Standardized Structure**: Every spec follows the same schema (`name`, `description`, `options?`, `args?`), enabling generic CLI tooling
