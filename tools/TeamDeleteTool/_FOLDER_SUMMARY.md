# Summary of `tools/TeamDeleteTool/`

## Purpose of `TeamDeleteTool/`

Implements the `TeamDelete` tool for the Claude swarm multi-agent system. This tool orchestrates the complete cleanup of a team and its associated resources when swarm work is finished, ensuring directories, session state, UI colors, and app state are all properly torn down. The tool enforces a precondition that all non-lead teammates must be shut down via `requestShutdown` before deletion is allowed.

## Contents Overview

| File | Role |
|------|------|
| `constants.ts` | Exports the single string constant `TEAM_DELETE_TOOL_NAME = 'TeamDelete'` — the canonical identifier used for tool registration and invocation throughout the codebase. |
| `prompt.ts` | Defines `getPrompt()`, which returns a Markdown-formatted prompt describing what the tool does, its three operations (team dir, task dir, session context removal), the critical preconditions, and the auto-detection of team name from session context. |
| `TeamDeleteTool.ts` | The core implementation — a `Tool` definition built with `buildTool`, containing the logic in `call()` that reads the team file to check active members, performs atomic cleanup via `setAppState()`, and triggers directory, color, and session cleanup side effects. It is deferred (`shouldDefer: true`) and gated behind `isAgentSwarmsEnabled()`. |
| `UI.tsx` | Provides two React rendering functions: `renderToolUseMessage` (always displays the static message `'cleanup team: current'`) and `renderToolResultMessage` (always returns `null` to suppress tool result output, since batched shutdown messages cover the feedback). |

## How Files Relate to Each Other

```
constants.ts
   └── TEAM_DELETE_TOOL_NAME (string identifier)
           │
           ├──▶ TeamDeleteTool.ts ──► referenced as tool.name
           │        │
           │        ├──▶ prompt.ts ──► getPrompt() attached as tool.prompt
           │        │
           │        └──▶ UI.tsx ──► renderToolUseMessage / renderToolResultMessage
           │                         attached as tool.renderUseMessage / renderResultMessage
           │
           └──▶ imported by consumers outside this directory
                (e.g., tool registration, invocation dispatch)
```

- `constants.ts` is the single source of truth for the tool name, avoiding hardcoded strings across the codebase.
- `TeamDeleteTool.ts` imports the constant, the prompt getter, and the UI renderers to assemble the complete tool definition.
- `prompt.ts` is purely functional — returns a string, no side effects, no imports.
- `UI.tsx` is a leaf module consumed only by the tool builder in `TeamDeleteTool.ts`.

## Key Takeaways

1. **Precondition enforcement** — The tool explicitly checks for active non-lead members and blocks deletion until `requestShutdown` has been called for all of them. Only idle/crashed members (`isActive === false`) are permitted to coexist with a deletion.
2. **Atomic state cleanup** — All app state mutations (clearing `teamContext`, `inbox.messages`, teammate colors, and leader name) happen in a single `setAppState()` call, ensuring consistent state after cleanup.
3. **Side-effect driven** — Actual filesystem directories are cleaned via `cleanupTeamDirectories()` and session registration is undone via `unregisterTeamForSessionCleanup()`.
4. **Feature-gated** — The tool is deferred and only enabled when `isAgentSwarmsEnabled()` returns true, ensuring it does not operate outside the swarm context.
5. **Silent completion** — User-facing output is suppressed (`null`); the tool "succeeds quietly" and relies on batched shutdown messages to communicate status to the user.
6. **Minimal public surface** — Four files total, with clear separation of concerns: naming, documentation, logic, and rendering.
