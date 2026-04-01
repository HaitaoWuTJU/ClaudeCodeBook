# Summary of `tools/TeamCreateTool/`

## Purpose of `TeamCreateTool/`

This directory implements a single Claude Code tool called **"TeamCreate"** that enables agents to create and manage multi-agent teams for coordinated parallel work. The tool encapsulates all logic, documentation, UI rendering, and configuration needed for spawning team structures with a designated leader agent and task coordination infrastructure.

## Contents Overview

| File | Role |
|------|------|
| `constants.ts` | Defines the canonical tool name constant (`TEAM_CREATE_TOOL_NAME = 'TeamCreate'`) |
| `prompt.ts` | Provides detailed agent-facing instructions on team creation, workflow, task ownership, message delivery, and team coordination patterns |
| `TeamCreateTool.ts` | Core implementation: input validation (Zod schema), business logic (file I/O, state management, analytics), and tool lifecycle hooks |
| `UI.tsx` | Renders a user-friendly message showing which team was created |
| `prompt.md` | Human-readable documentation for users |

## How Files Relate to Each Other

```
                    ┌─────────────────────────┐
                    │     constants.ts        │
                    │  (TEAM_CREATE_TOOL_NAME)│
                    └───────────┬─────────────┘
                                │
                                ▼
┌──────────────┐    ┌─────────────────────────┐    ┌──────────────┐
│   prompt.md  │◄───│    TeamCreateTool.ts    │───►│     UI.tsx   │
│ (reference)  │    │  (main tool definition) │    │  (rendering) │
└──────────────┘    └───────────┬─────────────┘    └──────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │       prompt.ts         │
                    │ (getPrompt() function)  │
                    └─────────────────────────┘
```

**Data flow:**

1. `TeamCreateTool.ts` imports `TEAM_CREATE_TOOL_NAME` from `constants.ts` to identify the tool
2. `TeamCreateTool.ts` imports `getPrompt()` from `prompt.ts` to provide instructions to the agent
3. `TeamCreateTool.ts` imports `renderToolUseMessage` from `UI.tsx` for UI output after execution
4. `UI.tsx` imports the `Input` type from `TeamCreateTool.ts` to properly type its props

**Key integration points:**

- `TeamCreateTool.ts` orchestrates the entire flow: validates input → creates team file → updates state → fires analytics → returns output to `UI.tsx` for rendering
- `prompt.ts` serves as both runtime instructions (fed to the LLM) and source for `prompt.md` documentation
- All cross-file type references go through `TeamCreateTool.ts`, which acts as the central hub

## Key Takeaways

1. **Centralized Tool Definition**: `TeamCreateTool.ts` is the authoritative source for input schema (`Input`), tool behavior, and type definitions. All other files depend on it.

2. **Separation of Concerns**:
   - **Logic** lives in `TeamCreateTool.ts` (validation, file I/O, state)
   - **Documentation** lives in `prompt.ts` (agent instructions) and `prompt.md` (user docs)
   - **Rendering** lives in `UI.tsx` (UI output only)

3. **Strict Input Validation**: Uses Zod v4 schema with `z.strictObject()` to ensure only documented fields are accepted (`team_name` required; `description`, `agent_type` optional).

4. **Session Cleanup**: Teams are registered via `registerTeamForSessionCleanup()` and have their task lists isolated, ensuring clean teardown when sessions end.

5. **Analytics Integration**: Fires a `tengu_team_created` event with metadata (team name, lead agent type, teammate mode) for tracking usage.

6. **Swarms Feature Gate**: The entire tool is disabled unless `isAgentSwarmsEnabled()` returns true, indicating this is part of an experimental or opt-in feature set.

7. **Deterministic Agent ID Generation**: The lead agent ID is computed as `formatAgentId(TEAM_LEAD_NAME, finalTeamName)`, ensuring reproducibility across sessions.

8. **Unique Name Collision Handling**: If a team name already exists on disk, `generateWordSlug()` automatically creates a unique alternative (e.g., `MyTeam` → `MyTeam-luminous-fox`).
