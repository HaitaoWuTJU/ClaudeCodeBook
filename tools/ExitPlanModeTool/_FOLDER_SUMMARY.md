# Summary of `tools/ExitPlanModeTool/`

## Purpose of `ExitPlanModeTool/`

Implements the Exit Plan Mode tool, which allows an AI (or teammate agent) to transition out of "plan mode" and begin coding. Plan mode is a state where the AI writes implementation plans for user/leader approval before executing code. This directory provides the core tool logic, permission handling, teammate approval workflows, and CLI UI rendering.

## Contents Overview

| File | Purpose |
|------|---------|
| `constants.ts` | Exports the tool name constant (`EXIT_PLAN_MODE_V2_TOOL_NAME`) |
| `prompt.ts` | Defines the LLM prompt instructing when/how to use the tool |
| `ExitPlanModeV2Tool.ts` | Core tool implementation with permission checks, state management, and teammate approval routing |
| `UI.tsx` | React/Ink UI components for rendering tool results in the CLI |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│  Prompt (prompt.ts)                                         │
│  Tells the AI: "Use this when you've finished your plan     │
│  and want user approval"                                    │
└─────────────────────────┬───────────────────────────────────┘
                          │ (injected as tool description)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  ExitPlanModeV2Tool.ts                                      │
│  • Reads plan from disk (getPlan)                           │
│  • Checks permissions (teammates bypass, users confirm)    │
│  • Routes teammate approvals to team-lead mailbox           │
│  • Transitions mode (plan → default/auto)                   │
│  • Handles auto-mode circuit breaker & permission stripping│
└──────────────┬──────────────────────────────────────────────┘
               │ references
               ▼
┌─────────────────────────────────────────────────────────────┐
│  constants.ts + UI.tsx                                      │
│  • Tool name constant                                       │
│  • Three UI states: empty plan, awaiting approval, approved │
└─────────────────────────────────────────────────────────────┘
```

**Flow Example — User exits plan mode:**
1. User runs `/exit-plan-mode`
2. LLM sees prompt → calls tool
3. Tool checks `isEnabled()` (fails for channels mode)
4. Tool validates `validateInput()` (must be in plan mode)
5. Tool calls `checkPermissions()` → shows confirmation UI
6. User approves → tool reads plan from disk
7. Tool transitions mode, writes updated plan (if CCR edited), updates state
8. `UI.tsx` renders the plan with `/plan to edit` hint

**Flow Example — Teammate exits plan mode:**
1. Teammate calls tool
2. `checkPermissions()` bypasses user UI (teammate)
3. If `isPlanModeRequired()` → sends mailbox message to team-lead
4. Teammate waits for inbox response
5. Team-lead approves → plan continues
6. Teammate exits plan mode

## Key Takeaways

1. **Dual paths for permission**: Teammates bypass the permission UI entirely and route approval to team-lead; non-teammates see a confirmation prompt.

2. **Feature-gated complexity**: Auto-mode (`autoModeStateModule`) and permission stripping (`permissionSetupModule`) are loaded conditionally via `feature('TRANSCRIPT_CLASSIFIER')`.

3. **Circuit breaker for auto mode**: If the user entered plan mode from auto mode but auto mode is now disabled (settings changed or circuit breaker triggered), the tool falls back to `default` mode.

4. **Permission lifecycle**: Permissions are stripped when entering plan mode from auto mode, and restored when exiting plan mode to non-auto modes.

5. **Three UI states**: The CLI renders differently based on: (a) empty plan, (b) awaiting team-lead approval, or (c) user-approved plan with Markdown rendering.

6. **Stub implementation**: `prompt.ts` is explicitly marked as a stub excluding "Ant-only" `allowedPrompts` content, suggesting an enterprise variant with additional semantic permission capabilities.
