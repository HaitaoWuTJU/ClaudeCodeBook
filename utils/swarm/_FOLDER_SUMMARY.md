# Summary of `utils/swarm/`

## Purpose of `utils/swarm/`

This directory implements the **Claude Swarm** multi-agent orchestration system. It enables a single "team lead" Claude instance to spawn, coordinate, and communicate with multiple "teammate" Claude instances that operate in parallel. The swarm can execute tasks collaboratively by distributing work across agents and aggregating results.

The system supports three execution models:
1. **tmux panes** — Teammates run in separate tmux windows (cross-platform)
2. **iTerm2 native panes** — Teammates run in iTerm2 split panes (macOS only)
3. **In-process** — Teammates run in the same Node.js process (AsyncLocalStorage-isolated)

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Public API entry point; exports `startTeam()` and `swarmSessionExists()` |
| `swarm.ts` | Core team orchestration (`startTeam()`, `joinTeam()`, `createTeammates()`) |
| `createAgentFromTeammateConfig.ts` | Transforms `TeammateConfig` → `AgentConfig` with system prompt composition |
| `teamHelpers.ts` | Team file I/O, mailbox creation, member management |
| `createTeammateMailbox.ts` | Creates per-teammate mailbox directories and files |
| `createTeammateInbox.ts` | Creates team inbox directory structure |
| `teammateLayoutManager.ts` | Assigns colors, creates/controls panes via backends |
| `teammateModel.ts` | Returns default Opus 4.6 model ID based on API provider |
| `teammatePromptAddendum.ts` | System prompt instructions for teammate communication |
| `inProcessRunner.ts` | In-process teammate execution loop (abortable, progress-tracked) |
| `leaderPermissionBridge.ts` | Queues permission requests to leader's UI dialog |
| `permissionSync.ts` | Mailbox-based permission request/response between leader and teammates |
| `tmuxUtils.ts` | tmux command execution and pane management utilities |
| `It2SetupPrompt.tsx` | Ink-based CLI wizard for installing/configuring iTerm2 split panes |
| `teammateHooks.ts` | Registers Stop hook to notify leader on teammate shutdown |
| `backends/` subdirectory | Pane/backend abstraction (see `backends/` summary above) |

## How Files Relate to Each Other

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         Public API (index.ts)                             │
│              startTeam() ◄────────────────────────────────────┐           │
│              swarmSessionExists()                               │           │
└────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         swarm.ts (Orchestrator)                           │
│                                                                             │
│  startTeam():           • Creates team inbox                               │
│                         • Spawns teammates via createTeammates()           │
│                         • Initializes layouts via teammateLayoutManager    │
│                                                                             │
│  joinTeam():           • Reads team config                                 │
│                        • Joins swarm session (tmux/iTerm2)                │
│                        • Starts mailbox polling loop                       │
│                                                                             │
│  createTeammates():    • Creates agent configs                             │
│                        • Registers with backends                           │
│                        • Spawns or starts in-process                       │
└────────────────────────┬───────────────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┬──────────────────┐
         ▼               ▼               ▼                  ▼
┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐ ┌──────────────────┐
│ teammateLayout   │ │ backends/   │ │ inProcessRunner │ │  teamHelpers     │
│ Manager         │ │ registry    │ │                 │ │                  │
│                 │ │             │ │                 │ │                  │
│ • assignColor() │ │ • getBest.. │ │ • runAgent()    │ │ • readTeamFile() │
│ • createPane()  │ │ • register  │ │ • abort ctrl    │ │ • writeTeamFile()│
│ • sendCommand() │ │             │ │ • progress      │ │ • mailbox ops    │
└────────┬────────┘ └──────┬──────┘ └────────┬────────┘ └────────┬─────────┘
         │                 │                 │                  │
         │          ┌──────▼──────┐          │                  │
         │          │   tmux.ts   │          │                  │
         │          │  iterm2.ts  │          │                  │
         │          │ InProcess.. │          │                  │
         │          └─────────────┘          │                  │
         │                                    │                  │
         │           ┌────────────────────────┴──────────────────┘
         │           │
         ▼           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    teammateMailbox.ts (Communication Hub)                  │
│                                                                             │
│  writeToMailbox()    writeToInbox()     createIdleNotification()           │
│  readMailbox()       waitForNextPromptOrShutdown()                         │
└────────────────────────────────────────────────────────────────────────────┘
```

**Prompt Composition Flow:**
```
teammatePromptAddendum.ts (static instructions)
    +
createAgentFromTeammateConfig.ts (appends teammate-specific system prompt)
    │
    ▼
teammateModel.ts (selects Opus 4.6 if no custom model)
    │
    ▼
teammateLayoutManager.ts (assigns color)
    │
    ▼
backends/ (spawns pane or in-process runner)
```

**Permission Flow:**
```
In-process teammate requests tool use
    │
    ▼
inProcessRunner.ts → createInProcessCanUseTool()
    │
    ├── Leader UI available ──► leaderPermissionBridge.ts ──► leader's ToolUseConfirmQueue
    │
    └── Leader UI unavailable ──► permissionSync.ts ──► Mailbox round-trip
```

## Key Takeaways

### Architecture

1. **Mailbox-centric communication**: All inter-agent messaging flows through file-based mailboxes (`~/.claude/swarm/teams/{teamId}/{agentId}/mailbox.json`). This decouples teammates from the leader's process, enabling true parallelism and crash isolation.

2. **Three-layer permission system**: Permissions for in-process teammates can resolve via three paths:
   - Leader's UI dialog (via `leaderPermissionBridge.ts`)
   - Mailbox round-trip (via `permissionSync.ts`)
   - Classifier auto-approval (for bash commands)

3. **Backend abstraction decouples orchestration from execution**: The `backends/` registry means the same swarm code works on Linux (tmux), macOS (tmux or iTerm2), and in-process without modification.

4. **Progress tracking per teammate**: In-process teammates use `createProgressTracker()` to emit structured progress updates to `AppState`, enabling real-time UI feedback without polling.

### Lifecycle

5. **AbortController dual-model**: In-process teammates support two cancellation granularities:
   - **Lifecycle abort** — terminates the entire teammate
   - **Work abort** — stops only the current turn, returns to idle (Escape key in UI)

6. **Cleanup on multiple exit paths**: Both pane-based and in-process backends implement cleanup:
   - Pane backends: `kill-pane` or `it2 session kill`
   - In-process: `lifecycleAbortController.abort()` + `currentWorkAbortController.abort()`
   - All paths: `evictTaskOutput()` + `markMemberInactive()` + idle notification

### Multi-Agent Coordination

7. **Round-robin color assignment**: `teammateLayoutManager` cycles through `AGENT_COLORS` to visually distinguish teammates in the UI. Assignments persist across a session.

8. **Team file as source of truth**: `teamHelpers.ts` manages a JSON team file that records the leader, all members, allowed paths, and inbox path. Teammates read this to locate the leader's mailbox.

9. **Inbox fan-out**: The leader's `createTeammates()` writes each teammate's inbox path to `inbox.json`, enabling the leader to iterate teammates and distribute work without re-reading the team file.

### Environment Handling

10. **tmux session modes**: The tmux backend adapts to two scenarios:
    - Leader already in tmux → spawns panes in existing session (30/70 leader/teammate split)
    - No tmux → creates dedicated `claude-swarm` session with `tiled` layout

11. **Shell init delay**: A 200ms delay after pane creation ensures shell RC files (starship, oh-my-zsh) load before commands execute, preventing race conditions.

12. **Provider-aware defaults**: `teammateModel.ts` maps API provider (Bedrock, Vertex, Foundry) → model ID, ensuring teammates use the correct Opus 4.6 variant for the user's infrastructure.

### Design Patterns

13. **Auto-registration side effects**: Each backend module calls `registerXxxBackend()` at module load time, avoiding circular dependencies while ensuring all backends are available when `getBestAvailableBackend()` is called.

14. **Cached detection at module load**: Environment detection (`isInsideTmux`, command availability) happens once at import time and is cached, because environment state cannot change during a process lifetime.

15. **AsyncLocalStorage for in-process isolation**: In-process teammates use `teammateContext.ts` and `agentContext.ts` with AsyncLocalStorage to isolate state (member IDs, callbacks, abort controllers) across concurrent in-process executions.
