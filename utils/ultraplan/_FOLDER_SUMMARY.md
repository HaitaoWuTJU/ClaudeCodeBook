# Summary of `utils/ultraplan/`

## Purpose of `ultraplan/`

Provides infrastructure for the `/ultraplan` (and `/ultrareview`) workflow — a two-phase process where a plan is first authored and approved in a remote browser, then optionally executed back in the local terminal. The directory is split into two concerns: **keyword detection** (routing user intent to the feature) and **session polling** (fetching the approved plan from the remote CCR environment).

## Contents Overview

| File | Role |
|------|------|
| `keyword.ts` | Standalone utility. Detects "ultraplan" / "ultrareview" triggers in raw text with context-aware filtering (skips paths, slash commands, quoted strings, etc.). Also exposes `replaceUltraplanKeyword()` to grammatically normalize prompts. |
| `ccrSession.ts` | Core polling engine. Opens a CCR session, continuously reads remote events via `pollRemoteSessionEvents`, classifies ExitPlanMode tool results, and returns `{ plan, rejectCount, executionTarget }` when the plan is approved or the user elects to teleport back. |

## How Files Relate to Each Other

```
User types "let's ultraplan this"
        │
        ▼
  keyword.ts
  hasUltraplanKeyword(text) ──► true
        │
        ▼
  replaceUltraplanKeyword(text) ──► "let's plan this"
        │
        ▼
  Prompt forwarded to agent
        │
        ▼
  Agent calls ExitPlanModeTool (remote CCR)
        │
        ▼
  ccrSession.ts
  pollForApprovedExitPlanMode(sessionId, ...) ──► blocks until verdict
        │
        ├── approved ──► plan text + executionTarget: 'remote'
        ├── teleport  ──► plan text + executionTarget: 'local'
        └── terminated ──► throws UltraplanPollError
```

`keyword.ts` is a **pre-filter** that decides whether to route the user to the ultraplan flow. `ccrSession.ts` is the **runtime engine** that executes that flow by coordinating with the remote CCR session.

## Key Takeaways

1. **Two-tier detection**: `hasUltraplanKeyword()` / `hasUltrareviewKeyword()` are gatekeepers; they are intentionally conservative to avoid false positives (e.g., `src/ultraplan/foo.ts`, `--ultraplan-mode`).

2. **Teleport sentinel**: The special string `__ULTRAPLAN_TELEPORT_LOCAL__` is injected by the browser UI when the user clicks "teleport back to terminal". This is the only path that sets `executionTarget: 'local'`.

3. **Stateful event classification**: `ExitPlanModeScanner` is a push-based reducer — it accumulates `SDKMessage[]` batches and emits a single `ScanResult`, handling interleaving of tool calls, tool results, and errors.

4. **Phase-driven polling**: The polling loop tracks `UltralanPhase` (`running` → `plan_ready` → `needs_input`) and only resolves when a terminal verdict (`approved` / `teleport` / `terminated`) is reached.

5. **Grammatical normalization**: `replaceUltraplanKeyword()` strips the keyword while preserving casing, so prompts forwarded to the model read naturally (e.g., "run ultraplan" → "run plan").
