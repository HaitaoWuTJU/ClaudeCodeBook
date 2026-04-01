# Summary of `components/agents/new-agent-creation/wizard-steps/`

## Purpose of `wizard-steps/`

This directory contains the **step components** that make up the multi-step wizard for creating new agents. Each step collects a specific piece of configuration (type, model, tools, memory, etc.) and passes data through a shared wizard context. The directory also includes the wizard orchestrators (`NewAgentWizard`, `CreateAgentWithWantedTasks`), the entry dialog (`AgentCreationDialog`), and a save wrapper (`ConfirmStepWrapper`).

## Contents Overview

| File | Purpose |
|------|---------|
| `TypeStep.tsx` | Collects the agent's unique identifier/slug (e.g., `"test-runner"`) |
| `WhenToUseStep.tsx` | Collects the task description / "when to use" text (freeform textarea) |
| `ModelStep.tsx` | Presents an `OptionPicker` with available AI models to choose from |
| `SystemPromptStep.tsx` | Collects the system prompt via `SystemPromptInput` (structured prompt builder) |
| `PromptStep.tsx` | Collects the system prompt via a simple `TextInput` (alternative to `SystemPromptStep`) |
| `ToolsStep.tsx` | Renders a `ToolSelector` to pick which tools the agent can access |
| `MemoryStep.tsx` | Renders a `MemorySelector` to configure agent memory settings |
| `ColorStep.tsx` | Renders a `ColorPicker` for the agent's background color |
| `LocationStep.tsx` | Collects the agent file directory via `LocationSelector` |
| `SummaryStep.tsx` | Shows a read-only summary of all collected data and handles final navigation |
| `ConfirmStep.tsx` | Displays a formatted summary with keyboard shortcuts for save/save-and-edit |
| `ConfirmStepWrapper.tsx` | Handles the actual file I/O: saves to disk, updates app state, optionally opens in editor, fires analytics |
| `NewAgentWizard.tsx` | Orchestrates all steps (`TypeStep` → `WhenToUseStep` → `ModelStep` → … → `SummaryStep`) |
| `CreateAgentWithWantedTasks.tsx` | Alternative simplified wizard (`TypeStep` → `SystemPromptStep` → `ToolsStep` → `SummaryStep`) |
| `AgentCreationDialog.tsx` | Entry point: renders the `NewAgentWizard` inside a `AgentCreationDialogLayout` |

## How Files Relate to Each Other

```
AgentCreationDialog.tsx
        │
        ▼
┌───────────────────────────────┐
│      NewAgentWizard.tsx       │   (or CreateAgentWithWantedTasks.tsx)
│  ┌─────────────────────────┐  │
│  │  useWizard<AgentWizardData>  │  ← shared context: wizardData, goNext/goBack
│  └──────────┬──────────────┘  │
│             │ step-by-step    │
│  ┌──────────┴──────────────┐  │
│  │   Step components       │  │
│  │                         │  │
│  │  TypeStep               │  │
│  │    └─→ wizardData.type  │  │
│  │  WhenToUseStep          │  │
│  │    └─→ wizardData.whenToUse  │
│  │  ModelStep              │  │
│  │    └─→ wizardData.selectedModel │
│  │  SystemPromptStep / PromptStep  │
│  │    └─→ wizardData.systemPrompt   │
│  │  ToolsStep              │  │
│  │    └─→ wizardData.selectedTools  │
│  │  MemoryStep             │  │
│  │    └─→ wizardData.memorySettings │
│  │  ColorStep              │  │
│  │    └─→ wizardData.selectedColor │
│  │  LocationStep           │  │
│  │    └─→ wizardData.location │
│  │                         │  │
│  │  SummaryStep ──────────────► renders summary, calls goNext()
│  │    └─→ wizardData.finalAgent (built on-the-fly)
│  │                         │  │
│  │  ConfirmStep            │  │
│  │    └─→ ConfirmStepWrapper.tsx (save logic) │
│  │        ├── saveAgentToFile()
│  │        ├── useSetAppState()
│  │        ├── editFileInEditor()
│  │        ├── logEvent()   │
│  │        └── onComplete() │
│  └─────────────────────────┘  │
└───────────────────────────────┘
```

1. **`AgentCreationDialog`** is the top-level entry point. It instantiates the wizard and manages `isActive`.
2. **`NewAgentWizard`** or **`CreateAgentWithWantedTasks`** iterate over their respective step components, sharing a **`useWizard`** context.
3. Each **step component** reads from and writes to the shared `wizardData` object. They only handle UI and local state (input fields, cursor positions, keyboard shortcuts).
4. **`SummaryStep`** assembles the data into a display-friendly summary.
5. **`ConfirmStep`** provides keyboard shortcuts (save / save-and-edit) and validation display.
6. **`ConfirmStepWrapper`** performs the **side effects**: writing the agent file to disk, updating application state, optionally launching an editor, and firing analytics.
7. Most step components use the **React Compiler** (`_c` pattern) and share common design-system primitives (`WizardDialogLayout`, `Byline`, `KeyboardShortcutHint`).

## Key Takeaways

- **Shared state via `useWizard<AgentWizardData>`**: All steps read/write to a single `wizardData` object, allowing steps to be reordered or swapped (e.g., `SystemPromptStep` vs. `PromptStep`) without changing downstream logic.
- **Two wizard variants**: `NewAgentWizard` is the full-featured flow; `CreateAgentWithWantedTasks` is a shorter flow that skips `ModelStep`, `MemoryStep`, and `ColorStep`.
- **Separation of UI and I/O**: Step components are purely presentational. `ConfirmStepWrapper` is the only component that performs file I/O and analytics, keeping side effects isolated.
- **Extensive keyboard navigation**: Every step provides `Enter` (confirm/continue), `↑↓` (navigate), and `Esc` (go back) shortcuts via `useKeybinding` and `ConfigurableShortcutHint`.
- **React Compiler optimization**: All step components use the `react/compiler-runtime` (`_c`, `Symbol.for("react.memo_cache_sentinel")`) pattern for fine-grained memoization of JSX subtrees and callbacks.
- **Context-locked keybindings**: The "confirm:no" binding uses `{ context: "Settings" }` in `TypeStep`, `WhenToUseStep`, and `PromptStep` so that typing `n` inside text inputs is not intercepted as a "no" action.
- **Validation on submit**: `TypeStep`, `WhenToUseStep`, `PromptStep`, and `SummaryStep` each run validation functions before advancing, displaying inline error messages when constraints are violated.
- **`finalAgent` assembly**: Steps like `ColorStep` and `SummaryStep` construct the `finalAgent` object on-demand from accumulated wizard data, normalizing optional fields (`selectedModel`, `selectedColor`) before the save step.
