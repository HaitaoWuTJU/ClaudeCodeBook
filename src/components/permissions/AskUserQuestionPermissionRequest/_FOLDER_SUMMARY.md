# Summary of `components/permissions/AskUserQuestionPermissionRequest/`

## Purpose of `AskUserQuestionPermissionRequest/`

This directory implements a permission request component for the Claude CLI that presents interactive questionnaires to users. When Claude needs to ask the user multiple questions before executing a tool, this module renders a terminal-based UI for collecting answers, handling multi-select options, text input, image attachments, and keyboard navigation.

## Contents Overview

| File | Role |
|------|------|
| `index.tsx` | Entry point; exports `AskUserQuestionPermissionRequest` which conditionally renders with/without syntax highlighting |
| `AskUserQuestionPermissionRequest.tsx` | Main container component with state management, analytics, image handling, and layout calculations |
| `use-multiple-choice-state.ts` | Custom hook (`useMultipleChoiceState`) managing questionnaire state: navigation, answers, per-question states |
| `PreviewBox.tsx` | Reusable bordered box component for rendering preview content with optional syntax highlighting and ANSI support |
| `PreviewQuestionView.tsx` | Side-by-side question layout with option list (left) and preview panel (right) |
| `SubmitQuestionsView.tsx` | Review screen showing all answered questions before final submission |
| `ask-user-question-tool.ts` | Zod schema definitions for question validation |
| `PermissionRequestTitle.tsx` | Styled title component for permission requests |
| `PermissionRuleExplanation.tsx` | Component explaining permission rules/outcomes |
| `QuestionNavigationBar.tsx` | Horizontal progress bar showing question navigation |

## How Files Relate to Each Other

```
AskUserQuestionPermissionRequest (index.tsx)
    │
    ├── useMultipleChoiceState (use-multiple-choice-state.ts)
    │       └── State: currentQuestionIndex, answers, questionStates
    │
    ├── AskUserQuestionPermissionRequest.tsx (orchestrator)
    │       │
    │       ├── Schema parsing: AskUserQuestionTool.inputSchema.safeParse()
    │       │
    │       ├── Layout calculations:
    │       │   └── maxHeight = terminalRows - CONTENT_CHROME_OVERHEAD
    │       │
    │       ├── Question rendering (conditional):
    │       │   ├── hasAnyPreview = true
    │       │   │       └── PreviewQuestionView.tsx
    │       │   │               └── PreviewBox.tsx (preview panel)
    │       │   │
    │       │   └── hasAnyPreview = false
    │       │           └── QuestionView (inline Select/SelectMulti)
    │       │
    │       ├── Submit view:
    │       │       └── SubmitQuestionsView.tsx
    │       │               ├── PermissionRuleExplanation.tsx
    │       │               └── PermissionRequestTitle.tsx
    │       │
    │       ├── Navigation:
    │       │       └── QuestionNavigationBar.tsx
    │       │
    │       ├── Image handling:
    │       │       ├── imageResizer → dimensions
    │       │       ├── imageStore → path caching
    │       │       └── convertImagesToBlocks → submission format
    │       │
    │       └── Analytics:
    │               ├── ask_user_question_accepted
    │               ├── ask_user_question_rejected
    │               ├── ask_user_question_respond_to_claude
    │               └── ask_user_question_finish_plan_interview
```

## Key Takeaways

1. **Dual rendering paths**: The component checks `syntaxHighlightingDisabled` in settings to decide between a basic layout or one with `PreviewBox` panels and syntax highlighting via `Suspense`.

2. **Question types**: Supports single-select (`layout="compact-vertical"`) and multi-select questions, with an auto-appended "Other" option featuring text input and external editor integration (`ctrl+g`).

3. **Plan mode support**: Special "Skip interview and plan immediately" footer option when in plan mode, plus `handleFinishPlanInterview` for early interview completion.

4. **Image support**: Full clipboard paste handling with `imageResizer` for downsampling and `imageStore` for path management; images are converted to `ImageBlockParam` before submission.

5. **Responsive layout**: Calculates content dimensions based on terminal size (`useTerminalSize`) with constants `MIN_CONTENT_HEIGHT=12`, `MIN_CONTENT_WIDTH=40`, `CONTENT_CHROME_OVERHEAD=15`.

6. **React compiler optimization**: All components use `react/compiler-runtime` with large cache arrays (`$` slots) for fine-grained memoization of computed values.

7. **State architecture**: Centralized in `useMultipleChoiceState` via `useReducer`, with state keyed by question text for order-agnostic storage.
