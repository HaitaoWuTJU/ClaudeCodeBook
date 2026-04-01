# Summary of `tools/shared/`

## Purpose of `shared/`

The `shared/` directory contains **cross-cutting utility modules** for the Claude Code tools system. It provides two main capabilities:

1. **Git operation tracking** — Detect git commits, pushes, and PR creations in command strings for usage analytics and OTLP metrics
2. **Multi-agent teammate spawning** — Unified infrastructure for creating teammate agents in-process, via tmux, or via iTerm2, extracted from `TeammateTool` for reuse by `AgentTool`

## Contents Overview

| File | Responsibility | Entry Points |
|------|----------------|--------------|
| `gitOperationTracking.ts` | Parse commands for git/gh/glab/curl operations, increment OTLP counters, fire analytics events, link sessions to PRs | `detectGitOperation()`, `trackGitOperations()` |
| `spawnMultiAgent.ts` | Spawn teammates as in-process agents or external panes, handle backend detection/fallback, manage team files, register background tasks | `spawnTeammate()`, `resolveTeammateModel()` |

## How Files Relate to Each Other

```
tools/shared/
├── gitOperationTracking.ts          # Metrics for git operations
└── spawnMultiAgent.ts               # Teammate lifecycle management
         │
         │ (imports from)
         ▼
    utils/swarm/                     # Shared swarm infrastructure
    ├── backends/                    # tmux/iTerm2/in-process detection & fallback
    ├── teamHelpers.js               # Team file I/O, name sanitization
    ├── teammateLayoutManager.js     # Color assignment, pane layout
    └── teammateModel.js              # Model resolution for teammates
```

**No direct coupling** exists between the two files—each is used independently by its respective consumer (`gitOperationTracking` by command execution hooks, `spawnMultiAgent` by `TeammateTool`/`AgentTool`). They share indirect ancestry through `utils/swarm/`.

## Key Takeaways

1. **Shell-agnostic design**: Both modules operate on raw command strings and parse output text rather than relying on shell-specific parsing, ensuring consistent behavior across Bash, PowerShell, and other shells.

2. **Analytics-first**: Git operation tracking feeds usage metrics (OTLP counters + analytics events) so the team can observe how collaborators use git workflows within Claude Code.

3. **Multi-backend spawning**: Teammate spawning detects available backends (tmux > iTerm2 > in-process), with graceful fallback—avoids hard failures when a pane backend is unavailable.

4. **Circular dependency guards**: `gitOperationTracking.ts` uses dynamic imports for `sessionStorage` to break import cycles; `spawnMultiAgent.ts` similarly avoids importing from teammate code that might transitively depend on it.

5. **Permission mode propagation**: Spawned teammates inherit the leader's permission mode (`bypassPermissions`, `acceptEdits`, `auto`), with `plan_mode_required` taking precedence to prevent bypass flags from leaking.
