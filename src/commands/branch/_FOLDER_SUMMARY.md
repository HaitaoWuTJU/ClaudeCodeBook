# Summary of `commands/branch/`

## Purpose of `branch/`

Implements the **`/branch`** command, which creates forks of the current conversation by copying transcript entries to a new session file. Each fork preserves the full conversation state (including content replacement records) while assigning a new session ID and maintaining traceability back to the original conversation. Users can optionally provide a custom name for the branch.

## Contents Overview

| File | Role | Key Responsibility |
|------|------|---------------------|
| `index.ts` | Command definition | Exports a `Command` object with metadata (`name`, `aliases`, `description`) and a lazy `load` function that imports the actual implementation |
| `branch.ts` | Command implementation | Contains the core logic: reading transcripts, filtering entries, generating unique branch names, writing fork files, and resuming into the branched session |

## How Files Relate to Each Other

```
index.ts (entry point)
│
├── Exports: Command definition object
│   ├── name: "branch"
│   ├── aliases: ["fork"] (conditional)
│   └── load: () => import('./branch.js')
│
└── On command invocation:
    │
    └──▶ branch.ts (implementation)
        │
        ├── deriveFirstPrompt() ────────────────▶ Extracts title from first user message
        ├── getUniqueForkName() ────────────────▶ Generates collision-safe names
        ├── createFork() ──────────────────────▶ Core logic: read → filter → rewrite → write
        │
        └── call() ────────────────────────────▶ Orchestrates fork creation + resume
```

**Data flow within `branch.ts`:**

1. **Read**: Current transcript file → parse JSONL entries → filter main conversation (exclude sidechains)
2. **Transform**: Clone entries with new session ID → add `forkedFrom` metadata → preserve content replacements
3. **Write**: New fork transcript file (mode `0o600`) → custom title to session storage → analytics event → resume hint

## Key Takeaways

- **Branching creates a complete snapshot**: Both conversation entries and content-replacement records are copied, ensuring the fork is self-contained and resumable with `claude -r {forkId}` without missing state.
- **Unique naming with pattern matching**: Branch names follow `"BaseName (Branch N)"` where N is derived by parsing existing session titles with regex, preventing name collisions across multiple forks.
- **Sidechains are excluded**: Only the main conversation thread is forked; auxiliary sidechain entries are filtered out, keeping the fork focused on the primary dialogue.
- **Lazy loading via `load` function**: The heavy implementation (`branch.ts`) is only imported when the command is invoked, keeping the command registry lightweight.
- **Conditional alias**: The `fork` alias exists only when `FORK_SUBAGENT` feature flag is disabled, avoiding conflicts with a potential standalone `/fork` command.
- **Analytics integration**: Tracks fork creation via `tengu_conversation_forked` events for metrics on conversation branching behavior.
