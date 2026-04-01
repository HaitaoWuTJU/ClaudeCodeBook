# Summary of `components/teams/`

## Purpose of `components/teams/`

The `teams/` directory contains React components that form the user interface for managing teammates within a team-based workflow. These components provide both a full-featured interactive dialog (`TeamsDialog`) and a lightweight status indicator (`TeamStatus`), enabling users to monitor, navigate, and control teammates through keyboard-driven terminal UI.

## Contents Overview

| File | Purpose |
|------|---------|
| `TeamsDialog.tsx` | Full interactive dialog for teammate management—list/detail views, kill, shutdown, hide/show, mode cycling, and output viewing via tmux |
| `TeamStatus.tsx` | Minimal footer component showing teammate count with conditional hint text |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                     AppState                            │
│              (teamContext.teammates)                    │
└───────────────────────┬─────────────────────────────────┘
                        │ read
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
  ┌───────────────┐          ┌───────────────┐
  │ TeamsDialog   │          │ TeamStatus    │
  │               │          │               │
  │ - Lists all   │          │ - Counts all  │
  │   teammates   │          │   teammates   │
  │ - Detail view │          │ - Shows hint  │
  │ - Actions     │          │               │
  └───────┬───────┘          └───────────────┘
          │
          ├──► tmux backend (viewTeammateOutput)
          ├──► mailbox IPC (sendShutdownRequestToMailbox)
          ├──► AppState (addHiddenPaneId, setMemberMode)
          └──► Backend registry (hide/show support)
```

**Data flow**: Both components read from `useAppState().teamContext.teammates` but serve different roles:
- `TeamStatus` is a passive display component (read-only)
- `TeamsDialog` is an active controller that reads statuses, polls for updates, and writes changes back to app state and the tmux backend

**Shared patterns**:
- Both use Ink's `Text` component
- Both use the React compiler (`_c` pattern with memoization arrays)
- `TeamsDialog` provides the interactive entry point for teammate actions; `TeamStatus` provides ambient awareness

## Key Takeaways

1. **Two-level UI pattern**: `TeamsDialog` implements a list/detail navigation pattern (`dialogLevel` state) to balance overview and focus.

2. **Periodic refresh**: The dialog polls `getTeammateStatuses` every 1 second to keep the teammate list current without manual refresh.

3. **Backend abstraction**: Hide/show functionality is gated on `getCachedBackend()?.supportsHideShow`, allowing graceful degradation for non-tmux backends.

4. **IPC-based shutdown**: Graceful teammate shutdown uses mailbox messaging (`sendShutdownRequestToMailbox`) rather than direct process control, decoupling the UI from process lifecycle.

5. **Tmux-centric design**: Output viewing (`viewTeammateOutput`) and pane management (`addHiddenPaneId`) assume a tmux backend, reflecting the project's terminal-native architecture.

6. **Permission mode cycling**: The component supports cycling individual and bulk permission modes via `setMemberMode` / `setMultipleMemberModes`, with `getNextPermissionMode` determining the next state.

7. **Keyboard-driven**: All interactions (navigation, actions, mode cycling) are keyboard-only, fitting the Ink terminal UI paradigm.

8. **React compiler usage**: Both files use the React compiler with manual cache arrays (`$`) for fine-grained memoization of sub-components (`TeamDetailView`, `TeammateListItem`, `TeammateDetailView`).
