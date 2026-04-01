# Summary of `tools/ExitWorktreeTool/`

## Purpose of `ExitWorktreeTool/`

The `ExitWorktreeTool/` directory implements a git worktree exit mechanism for the Tengu CLI tool. Its purpose is to cleanly exit a git worktree session that was previously created by `EnterWorktreeTool`, restoring the original working directory, project root, and system state while optionally preserving or removing the worktree directory itself.

This tool is part of a broader git worktree management workflow that allows developers to create isolated working directories for different tasks while maintaining a clean, organized development environment.

## Contents Overview

| File | Purpose |
|------|---------|
| `constants.ts` | Defines the string constant `EXIT_WORKTREE_TOOL_NAME = 'ExitWorktree'` for tool identification |
| `ExitWorktreeTool.ts` | Core implementation with Zod schemas, validation logic, change counting, session restoration, and git operations |
| `prompt.ts` | Generates a detailed documentation prompt describing tool scope, parameters, and behavior |
| `UI.tsx` | React components (Ink.js) for rendering "tool use" and "tool result" messages in the terminal |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                          ENTRY POINTS                               │
│                                                                     │
│  Index consumers import:                                            │
│    • EXIT_WORKTREE_TOOL_NAME  →  constants.ts                       │
│    • ExitWorktreeTool         →  ExitWorktreeTool.ts                │
│    • getExitWorktreeToolPrompt →  prompt.ts                         │
│    • UI components             →  UI.tsx                             │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     INTERNAL DEPENDENCIES                           │
│                                                                     │
│  ExitWorktreeTool.ts orchestrates everything:                       │
│    1. References EXIT_WORKTREE_TOOL_NAME (constants.ts)             │
│    2. Uses lazySchema for input/output Zod schemas                  │
│    3. Calls getExitWorktreeToolPrompt() for tool documentation      │
│    4. Uses renderToolUseMessage/renderToolResultMessage (UI.tsx)    │
│    5. Imports utilities from:                                       │
│       • worktree.js (getCurrentWorktreeSession, keepWorktree,        │
│         cleanupWorktree, killTmuxSession)                            │
│       • state.js (cwd/project root management)                      │
│       • hooks, memory caches, plans directory                        │
│       • analytics (logging)                                         │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Two-Mode Operation**: The tool supports two actions — `"keep"` (preserves the worktree directory and branch) and `"remove"` (deletes the worktree entirely). The `discard_changes` flag allows forced removal even with uncommitted changes after user confirmation.

2. **Fail-Closed Safety**: The tool uses a fail-closed approach when counting uncommitted changes. If git commands fail, it assumes unsafe conditions and blocks the operation to prevent accidental data loss.

3. **Session Restoration**: Upon exit, the tool reverses the effects of `EnterWorktreeTool` by restoring the original working directory, project root, system prompt sections, memory caches, and plans directory — ensuring a clean return to the main development context.

4. **Tmux Integration**: When a tmux session was created for the worktree, the tool can either preserve it (keeping the session alive for reattachment) or terminate it (on remove action).

5. **Scope Isolation**: The tool only operates on worktrees created within the current session by `EnterWorktreeTool`. It explicitly rejects manual worktrees or those from previous sessions, preventing unintended side effects.

6. **Analytics Tracking**: Both keep and remove actions are logged to analytics with commit/file counts and a `mid_session: true` flag, enabling tracking of worktree usage patterns.
