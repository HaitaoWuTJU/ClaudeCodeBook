# Summary of `commands/rewind/`

## Purpose of `commands/rewind/`

Implements the **`rewind` CLI command** (aliased as `checkpoint`) that allows users to restore code and/or conversation to a previous state.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command registration — exports a metadata object that lazily imports the implementation |
| `rewind.ts` | Command handler — the actual logic (currently a stub that returns `type: 'skip'`) |

## How Files Relate to Each Other

```
index.ts                    rewind.ts
┌──────────────────┐        ┌──────────────────┐
│ satisfies        │        │ export async     │
│   Command        │──lazy──│   function call() │
│ interface        │ import │                  │
└──────────────────┘        └──────────────────┘
```

1. `index.ts` defines command **metadata** (name, aliases, description, type) and satisfies the `Command` interface.
2. The `load` callback uses **dynamic import** (`() => import('./rewind.js')`) to defer loading of the actual implementation.
3. `rewind.ts` provides the `call()` function, which is the **runtime entry point** when the command executes.

## Key Takeaways

- **Lazy loading pattern**: The command implementation is not bundled on startup; it loads only when the user invokes `rewind`.
- **Stub implementation**: The current `rewind.ts` is a no-op placeholder (`type: 'skip'`) — the actual restore/checkpoint logic has not yet been implemented.
- **Local command type**: This command runs entirely on the client side (`type: 'local'`) and does not support non-interactive mode (`supportsNonInteractive: false`).
- **Command alias**: Users can invoke the command via `checkpoint` in addition to `rewind`.
