# Summary of `tools/EnterPlanModeTool/`

## Purpose of `EnterPlanModeTool/`

Provides a tool that allows a user (or the top-level AI assistant acting on behalf of a user) to request entry into **"plan mode"** — a read-only, exploratory phase for understanding a codebase and designing an implementation strategy before any code is written. It handles the full lifecycle of that request: gating, execution, state transition, and UI feedback.

---

## Contents Overview

| File | Role |
|------|------|
| `constants.ts` | Defines the canonical string identifier `'EnterPlanMode'` used everywhere the tool is referenced. |
| `EnterPlanModeTool.ts` | Core tool implementation: input/output schemas, `isEnabled` gate, `call` logic (state transition + permission update), and result mapping. |
| `prompt.ts` | Prompt instructions that tell the AI **when to invoke** the tool. Exports two variants (`External` / `Ant`) selected via `USER_TYPE` env var. |
| `UI.tsx` | Ink-based CLI rendering for the tool result and rejection messages. |
| `messages.ts` | *(referenced)* Plan mode workflow attachment content, injected when `isPlanModeInterviewPhaseEnabled` is true. |
| `index.ts` | *(referenced)* Module entry point that re-exports the assembled tool. |

---

## How Files Relate to Each Other

```
prompt.ts (tool-use instructions)
    │
    ▼
index.ts  ──►  EnterPlanModeTool.ts (assembled via buildTool)
                    │
                    ├── constants.ts  ──► used as tool identifier
                    ├── EnterPlanModeTool.ts::call()
                    │       │
                    │       ├── reads context.agentId  ──► blocks sub-agents
                    │       ├── reads context.appState.permission
                    │       ├── reads feature flags (KAIROS, KAIROS_CHANNELS)
                    │       ├── calls handlePlanModeTransition()
                    │       ├── calls prepareContextForPlanMode()
                    │       ├── calls applyPermissionUpdate({ type: 'setMode', mode: 'plan' })
                    │       └── calls context.setAppState()  ──► persisted to permission store
                    │
                    └── UI.tsx
                            ├── renderToolResultMessage()  ──► success view
                            └── renderToolUseRejectedMessage()  ──► decline view
```

**Dependency order**: `constants.ts` → `EnterPlanModeTool.ts` → `index.ts`. `prompt.ts` and `UI.tsx` are injected as configuration to `buildTool`.

---

## Key Takeaways

1. **Feature-gate on KAIROS**: The tool is **disabled** when `KAIROS` or `KAIROS_CHANNELS` flags are active **and** allowed channels exist. This prevents a UX trap where the model enters plan mode but cannot render the exit dialog.

2. **No sub-agents**: The `call()` method throws if `context.agentId` is set. Plan mode entry is reserved for the root user session only.

3. **Self-managed permissions**: `shouldDefer: true` means the tool handles its own permission flow — it transitions `permission.mode` to `'plan'` directly rather than requesting a separate approval dialog.

4. **Two prompt personas**: The `prompt.ts` exports differ significantly in tone:
   - **External**: Comprehensive (7 criteria), encourages planning early and often.
   - **Ant**: Concise (3 criteria), pushes the model to start working when the path is clear and only invoke plan mode for genuine ambiguity.

5. **Conditional workflow content**: When `isPlanModeInterviewPhaseEnabled()` is true, the detailed 6-step plan mode instructions live in `messages.ts` rather than in the success UI — keeping the result message lightweight.

6. **Pure read-only operation**: `isReadOnly: true` signals that invoking this tool does not constitute a write to the codebase, only a change to the assistant's operational mode.
