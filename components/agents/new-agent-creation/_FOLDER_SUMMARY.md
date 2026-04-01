# Summary of `components/agents/new-agent-creation/`

# `new-agent-creation/` — Agent Creation Wizard System

## Purpose of `new-agent-creation/`

This directory provides a complete, multi-step wizard UI for creating new agents within the application. It abstracts the complexity of agent creation into a guided, step-by-step experience where users select or input: a unique identifier, task description, AI model, system prompt, tools, memory settings, display color, and file location. The system is built with separation of concerns in mind — step components handle only UI and local state, while a dedicated wrapper handles the side effects of persisting the agent to disk and updating application state.

## Contents Overview

| File/Directory | Role |
|---|---|
| `types.ts` | TypeScript interfaces (`AgentWizardData`, `AgentFinalData`) shared across all wizard steps |
| `index.ts` | Public API; exports `CreateAgentWizard` and related types |
| `CreateAgentWizard.tsx` | Top-level orchestrator; builds the step list and wraps everything in `WizardProvider` |
| `dialogs/AgentCreationDialog.tsx` | Modal dialog entry point that renders the wizard with dialog chrome |
| `dialogs/AgentCreationDialogLayout.tsx` | Shared layout component for agent creation dialogs (Byline, keyboard shortcut hints) |
| `wizard-steps/` | Individual step components that each collect one piece of agent configuration |
| `wizard-steps/TypeStep.tsx` | Collects the agent's unique slug/identifier |
| `wizard-steps/WhenToUseStep.tsx` | Collects the task description / "when to use this agent" text |
| `wizard-steps/ModelStep.tsx` | Displays an `OptionPicker` of available AI models |
| `wizard-steps/SystemPromptStep.tsx` | Collects the system prompt via a structured prompt builder (`SystemPromptInput`) |
| `wizard-steps/PromptStep.tsx` | Alternative prompt step using a simple `TextInput` |
| `wizard-steps/ToolsStep.tsx` | Renders a `ToolSelector` for choosing which tools the agent may use |
| `wizard-steps/MemoryStep.tsx` | Renders a `MemorySelector` for configuring memory settings |
| `wizard-steps/ColorStep.tsx` | Renders a `ColorPicker` for the agent's background color |
| `wizard-steps/LocationStep.tsx` | Renders a `LocationSelector` for choosing the agent file directory |
| `wizard-steps/SummaryStep.tsx` | Displays a read-only summary of all collected data |
| `wizard-steps/ConfirmStep.tsx` | Shows a formatted summary with keyboard shortcuts for save/save-and-edit |
| `wizard-steps/ConfirmStepWrapper.tsx` | Handles all side effects: file I/O, app state updates, editor launch, analytics |
| `wizard-steps/NewAgentWizard.tsx` | Full wizard orchestrator (TypeStep → WhenToUseStep → ModelStep → … → SummaryStep) |
| `wizard-steps/CreateAgentWithWantedTasks.tsx` | Simplified wizard variant (TypeStep → SystemPromptStep → ToolsStep → SummaryStep) |

## How Files Relate to Each Other

```
AgentCreationDialog.tsx
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│              CreateAgentWizard.tsx                        │
│  • Conditionally includes MemoryStep if isAutoMemoryEnabled │
│  • Builds ordered steps[] array                            │
│  • Renders WizardProvider with steps, onCancel             │
└─────────────────────────┬─────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────────┐
│         NewAgentWizard.tsx  (or CreateAgentWithWantedTasks) │
│  • useWizard<AgentWizardData>() — shared wizard context     │
│  • Maps steps to WizardStepComponent wrappers               │
│  • Handles showStepCounter={false}                          │
└─────────────────────────┬─────────────────────────────────┘
                          │
              ┌───────────┴────────────────────────────────┐
              │          Step components (UI only)           │
              │                                             │
              │  TypeStep ──────────────── wizardData.type  │
              │  WhenToUseStep ─────────── wizardData.whenToUse
              │  ModelStep ─────────────── wizardData.selectedModel
              │  SystemPromptStep / PromptStep               │
              │    └─────────────────── wizardData.systemPrompt
              │  ToolsStep ────────────── wizardData.selectedTools
              │  MemoryStep ───────────── wizardData.memorySettings
              │  ColorStep ─────────────── wizardData.selectedColor
              │  LocationStep ──────────── wizardData.location │
              │                                             │
              │  SummaryStep ────────────────────────────────┐
              │    └─ assembles wizardData → finalAgent      │
              │                                             │
              │  ConfirmStep ────────────────────────────────┤
              │    └─ KeyboardShortcutHint (save / edit)     │
              │                                             │
              │  ConfirmStepWrapper.tsx ────────────────────┘
              │    ├─ saveAgentToFile(finalAgent)
              │    ├─ useSetAppState() (adds agent to app state)
              │    ├─ editFileInEditor() (optional)
              │    ├─ logEvent("agent_created", ...)
              │    └─ onComplete(message)
              └─────────────────────────────────────────────┘
```

**Data flow summary:**

1. **`AgentCreationDialog`** is the modal entry point. It owns `isActive` state and renders `CreateAgentWizard` when active.
2. **`CreateAgentWizard`** constructs the ordered `steps` array and provides it to `WizardProvider`. It conditionally appends `MemoryStep` based on the `isAutoMemoryEnabled()` feature flag.
3. **`NewAgentWizard`** (or `CreateAgentWithWantedTasks`) receives the steps and wraps each in a `WizardStepComponent`, which provides access to the shared `useWizard<AgentWizardData>` context.
4. **Each step component** reads from and writes to the shared `wizardData` object. They handle only local UI state (input values, cursor positions, keyboard shortcuts, validation messages).
5. **`SummaryStep`** assembles the accumulated data into a `finalAgent` object in real-time as the user navigates backward/forward.
6. **`ConfirmStep`** provides a last review screen with keyboard shortcuts (`Enter` to save, `Shift+Enter` to save and open editor).
7. **`ConfirmStepWrapper`** is the only component that performs side effects — it is not a visible step but wraps the final confirmation. It writes the agent file to disk, updates the application's agent list, optionally opens the file in an editor, fires analytics events, and calls `onComplete`.

## Key Takeaways

- **Two wizard variants:** `NewAgentWizard` is the full-featured flow (10+ steps including model selection, color, and optional memory). `CreateAgentWithWantedTasks` is a shorter, simplified flow that omits those steps and uses `SystemPromptStep` instead of `PromptStep`.

- **Shared `AgentWizardData` context:** All steps read/write to a single typed object via `useWizard<AgentWizardData>()`, enabling steps to be reordered, swapped, or conditionally included (like `MemoryStep`) without impacting downstream logic.

- **Clear separation of UI and I/O:** Step components are entirely presentational. `ConfirmStepWrapper` is the single point of side effects, keeping the codebase predictable and testable.

- **Extensive keyboard navigation:** Every step provides contextual keyboard shortcuts via `useKeybinding` and `ConfigurableShortcutHint` — `Enter` to confirm, `↑↓` to navigate, `Esc` to go back. The "confirm:no" binding uses `{ context: "Settings" }` in text-input steps so that typing `n` is not intercepted.

- **Validation on step advancement:** `TypeStep`, `WhenToUseStep`, `PromptStep`, and `SummaryStep` each run validation functions before calling `goNext()`, surfacing inline error messages when constraints are violated.

- **React Compiler optimization:** All step components use the `react/compiler-runtime` pattern (`_c`, `Symbol.for("react.memo_cache_sentinel")`) for fine-grained automatic memoization of JSX subtrees, reducing unnecessary re-renders in input-heavy UIs.

- **`finalAgent` is assembled on-demand:** Steps like `ColorStep` and `SummaryStep` construct the `finalAgent` object from accumulated `wizardData` at display time, normalizing optional fields (`selectedModel`, `selectedColor`) before the save step.

- **Feature-gated memory step:** `isAutoMemoryEnabled()` from `memdir/paths.js` determines at runtime whether `MemoryStep` appears in the wizard, allowing the memory feature to be shipped independently of the core wizard UI.

- **Analytics and telemetry:** `ConfirmStepWrapper` fires `logEvent("agent_created", ...)` with structured metadata (agent name, type, tool count, model, memory settings) for tracking usage patterns.

- **`LocationStep` handles file conflict resolution:** It prompts the user with the full agent file path and checks for existing files before the agent is saved, preventing accidental overwrites.
