# Summary of `tools/TaskGetTool/`

## Purpose of `TaskGetTool/`

A self-contained tool module that provides the **"Task Get"** functionality — allowing the system to retrieve detailed information about a specific task by its ID, including its status, description, and blocking relationships with other tasks.

---

## Contents Overview

| File | Purpose |
|------|---------|
| `constants.ts` | Defines the canonical tool name identifier (`TaskGet`) |
| `prompt.ts` | Contains the natural-language prompt template and human-readable description shown to the LLM/user |
| `TaskGetTool.ts` | Implements the core tool logic: input validation, task fetching, output formatting, and feature gating |

---

## How Files Relate to Each Other

```
constants.ts ─────┐
                 │─→ TaskGetTool.ts (uses both as configuration)
prompt.ts ───────┘
```

- **`constants.ts`** and **`prompt.ts`** are pure data modules with no dependencies — they exist to separate configuration from logic, making the tool easy to modify without touching the core implementation.

- **`TaskGetTool.ts`** is the orchestrator that:
  1. Imports `TASK_GET_TOOL_NAME` as the tool's identifier
  2. Imports `DESCRIPTION` and `PROMPT` to register the tool's metadata with the framework
  3. Defines input/output schemas using **Zod v4**
  4. Implements the `call()` method to fetch a real task via `getTask()`
  5. Implements `mapToolResultToToolResultBlockParam()` to render a human-friendly output message

---

## Key Takeaways

1. **Read-only tool** — `isReadOnly: true` means this tool never modifies system state; it only fetches and displays task data.
2. **Deferred execution** — `shouldDefer: true` means the tool is queued for execution rather than running immediately.
3. **Feature-gated** — The tool is only available when `isTodoV2Enabled()` returns `true`.
4. **Dependency-aware** — The tool surfaces blocking relationships (`blockedBy` and `blocks`), helping users understand task prerequisites.
5. **Clean separation** — Configuration (constants, prompts) is kept separate from implementation, following a common pattern used across other tool directories in this codebase.
