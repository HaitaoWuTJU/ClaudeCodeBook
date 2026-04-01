# Summary of `tools/EnterWorktreeTool/`

## Purpose of `EnterWorktreeTool/`

Provides a complete tool for creating isolated Git worktrees and switching the current Claude Code session into them mid-task. This enables parallel development contexts without losing the current work session.

## Contents Overview

| File | Role |
|------|------|
| `constants.ts` | Single export: `ENTER_WORKTREE_TOOL_NAME = 'EnterWorktree'` — canonical identifier used throughout the codebase |
| `EnterWorktreeTool.ts` | Core logic — input/output schemas, tool execution handler, validation, cache management |
| `prompt.ts` | Tool-usage documentation — instructs the AI assistant on when to activate and how to invoke the tool |
| `UI.tsx` | Terminal rendering — shows "Creating worktree…" during execution and renders the branch/path result |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────┐
│                    tool registration                      │
│                                                          │
│  constants.ts ────► EnterWorktreeTool.ts                 │
│                          │                               │
│                          ├──► uses inputSchema/outputSchema
│                          ├──► executes createWorktreeForSession()
│                          ├──► clears caches post-creation
│                          └──► emits analytics event      │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────┐
│                    runtime display                        │
│                                                          │
│  prompt.ts ────► consumed by AI for tool invocation       │
│                  guidance                                  │
│                                                          │
│  UI.tsx ────► renders terminal messages via Ink           │
│                  (progress + result Box/Text layout)      │
└──────────────────────────────────────────────────────────┘
```

1. **`constants.ts`** defines the tool name once, avoiding hardcoded strings across the codebase
2. **`EnterWorktreeTool.ts`** is the orchestrator — it declares schemas, validates input (via `validateWorktreeSlug`), creates the worktree, updates session state, and clears caches
3. **`prompt.ts`** provides a human-readable usage guide that likely gets embedded into the AI's system prompt
4. **`UI.tsx`** provides the terminal-rendered feedback using Ink's `Box` and `Text` components

## Key Takeaways

- **Mid-session worktree switching**: Unlike a pre-session initialization, this tool defers execution (`shouldDefer: true`) to create a worktree mid-conversation and immediately switch into it
- **Cache invalidation**: After worktree creation, multiple caches are explicitly cleared (system prompts, memory files, plans directory) to ensure the new context is clean
- **Slug validation**: Worktree names are validated against a custom regex (alphanumeric segments, dots, underscores, dashes; max 64 chars) using a Zod `superRefine`
- **Optional naming**: Users can specify a custom name or the tool falls back to the current plan's slug
- **Analytics tracking**: Emits `tengu_worktree_created` with `mid_session: true` to distinguish from pre-session worktree creation
