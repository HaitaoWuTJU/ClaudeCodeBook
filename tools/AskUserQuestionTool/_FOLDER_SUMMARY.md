# Summary of `tools/AskUserQuestionTool/`

## Purpose of `AskUserQuestionTool/`

This directory implements a CLI tool enabling an AI to present users with 1–4 multiple-choice questions through an interactive terminal UI, collect their answers, and return structured results for downstream use. It supports single and multi-select questions, optional HTML or Markdown preview content per option, and user annotations.

## Contents Overview

| File | Role |
|------|------|
| `prompt.ts` | Configuration constants, tool description, and detailed usage prompts. Exports tool name, chip width, description, preview feature instructions (Markdown + HTML variants), and usage guidelines including plan-mode warnings and "Other" option behavior. |
| `tool.ts` | Core tool implementation (`AskUserQuestionTool`). Defines Zod schemas for input (1–4 questions with uniqueness constraints) and output (answers map + annotations), implements all `Tool<ToolDef>` lifecycle methods (`isEnabled`, `validateInput`, `checkPermissions`, `call`, `mapToolResultToToolResultBlockParam`), and exports `_sdkInputSchema`/`_sdkOutputSchema` for external SDK consumers. |
| `index.ts` | Module entry point that re-exports all public symbols from both `tool.ts` and `prompt.ts`, making them available to the rest of the codebase. |

## How Files Relate to Each Other

```
prompt.ts  ─────────────────────────────────────────────────────────────┐
  Exports: NAME, DESCRIPTION, PREVIEW_FEATURE_PROMPT, USAGE_PROMPT     │
                                                                         │
tool.ts  ◄── imports NAME, DESCRIPTION, PREVIEW_FEATURE_PROMPT ────────┘
  Imports: _sdkInputSchema / _sdkOutputSchema
  Exports: AskUserQuestionTool, validateHtmlPreview, SDK schemas
                                                                         │
index.ts  ◄── re-exports everything from both files
```

- `prompt.ts` is the **pure configuration layer**: strings and constants only, no logic.
- `tool.ts` is the **logic layer**: consumes prompt constants, implements the tool contract, and exposes SDK-facing schemas.
- `index.ts` is the **public API surface**: aggregates and re-exports all symbols for consumers.

## Key Takeaways

- **TUI via Ink/React** — Answers are collected interactively in the terminal; no web UI required.
- **Multi-select capable** — When `multiSelect: true`, answers are comma-separated in the output.
- **HTML preview validation** — Fragments are validated: no `<html>`, `<body>`, `<script>`, or `<style>` tags; inline styles only.
- **Channel gating** — Tool is automatically disabled (`isEnabled() → false`) when KAIROS channels (Telegram/Discord) are active, preventing hangs.
- **SDK exportable** — `_sdkInputSchema` and `_sdkOutputSchema` allow external SDK consumers to configure tool behavior via `toolConfig.askUserQuestion`.
- **Uniqueness enforced** — Both question texts across the batch and option labels within each question must be unique.
- **Plan-mode safe** — Prompts explicitly warn against referencing "the plan" since users cannot see it until `EXIT_PLAN_MODE_TOOL_NAME` is called.
