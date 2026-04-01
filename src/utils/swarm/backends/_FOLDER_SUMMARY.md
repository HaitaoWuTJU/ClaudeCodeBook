# Summary of `utils/swarm/backends/`

## Purpose of `backends/`

The `backends/` directory implements the execution layer for the Claude Swarm multi-agent system. It provides abstraction over different terminal and process management technologies, enabling teammates to run in:

- **tmux panes** (cross-platform terminal multiplexer)
- **iTerm2 native panes** (macOS-only with Python API)
- **In-process** (same Node.js process, AsyncLocalStorage-isolated)

This design decouples the swarm orchestration logic from the specifics of how teammates are actually executed, allowing the system to adapt to the user's environment.

## Contents Overview

| File | Purpose |
|------|---------|
| `types.ts` | TypeScript interfaces (`TeammateExecutor`, `PaneBackend`, spawn/message/result types) |
| `constants.ts` | Shared configuration (socket names, session names, command paths) |
| `detection.ts` | Terminal detection (tmux/iTerm2 availability, running-in-tmux, pane IDs) |
| `registry.ts` | Backend registration and selection (`getBestAvailableBackend()`) |
| `tmux.ts` | tmux pane backend implementation |
| `iterm2.ts` | iTerm2 native pane backend implementation |
| `it2Setup.ts` | it2 CLI installation, verification, and user preferences |
| `InProcessBackend.ts` | In-process teammate execution (same-process, AsyncLocalStorage) |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────────┐
│                          TeammateTool                                │
│                    (orchestrates teammates)                          │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ getBestAvailableBackend()
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                           registry.ts                                │
│              Auto-registers all backends on import                   │
│         Exposes: getBestAvailableBackend(), getAllBackends()         │
└────────────┬─────────────────────────┬───────────────────────────────┘
             │                         │
    ┌────────▼──────────┐    ┌─────────▼─────────┐
    │   tmux.ts         │    │   iterm2.ts       │
    │ PaneBackend impl  │    │ PaneBackend impl  │
    │ • createPane()    │    │ • createPane()    │
    │ • killPane()      │    │ • killPane()      │
    │ • setBorderColor()│    │ • setBorderColor()│
    │ • hide/showPane() │    │ • rebalancePanes()│
    └────────┬──────────┘    └─────────┬─────────┘
             │                         │
             └───────────┬─────────────┘
                         │
              ┌──────────▼──────────┐
              │   types.ts          │
              │ PaneBackend interface│
              │ TeammateExecutor    │
              └─────────────────────┘
                         │
              ┌──────────▼──────────┐
              │  InProcessBackend   │
              │ TeammateExecutor impl│
              │ (no PaneBackend)    │
              └─────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                         detection.ts                                 │
│         Shared detection utilities (used by registry + backends)     │
│              isInsideTmux(), isTmuxAvailable(), isInITerm2()         │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                        it2Setup.ts                                   │
│    Dependency for iterm2.ts — handles CLI installation/verification  │
│              installIt2(), verifyIt2Setup(), preferences             │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       constants.ts                                   │
│              TMUX_COMMAND, socket names, session names               │
└──────────────────────────────────────────────────────────────────────┘
```

**Auto-registration pattern**: Each backend module executes `registerXxxBackend(XxxBackend)` as a side effect at module load time. This avoids circular dependencies while ensuring all backends are available when the registry is queried.

## Key Takeaways

1. **Dual-interface architecture**: `PaneBackend` handles low-level pane operations (create, kill, style), while `TeammateExecutor` provides high-level teammate lifecycle management. Pane-based backends implement both; `InProcessBackend` implements only `TeammateExecutor`.

2. **Environment-adaptive selection**: `registry.ts` uses `detection.ts` to determine the best available backend, preferring the user's native environment when teammates are started from within tmux/iTerm2.

3. **Two tmux session modes**: The tmux backend operates in two modes:
   - **With leader**: Spawns panes in the user's existing session alongside the leader (30/70 split via `main-vertical`)
   - **Without leader**: Creates a separate `claude-swarm` session with `tiled` layout

4. **In-process teammates share resources**: Unlike pane-based teammates, in-process teammates reuse the leader's API clients and MCP connections. They're isolated via `AsyncLocalStorage` but can hit rate limits or conflict on state.

5. **Mailbox-based communication**: All teammates use file-based mailboxes (`writeToMailbox()`) regardless of execution method, providing a consistent communication channel.

6. **AbortController for cancellation**: In-process teammates are cancelled via `AbortController.signal`, while pane-based teammates are killed via `kill-pane` or `it2 session kill`.

7. **Caching for stability**: Detection results are cached at module load time because environment state (TMUX env var, available commands) cannot change during a process lifetime.

8. **Shell initialization delay**: A 200ms delay after pane creation ensures shell config files load before commands are sent, preventing race conditions with starship/oh-my-zsh initialization.

9. **Leader pane ID preservation**: `detection.ts` captures `TMUX_PANE` at module load time *before* `Shell.ts` potentially modifies `process.env.TMUX`, ensuring commands always target the correct pane.
