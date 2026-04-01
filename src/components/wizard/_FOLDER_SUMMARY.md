# Summary of `components/wizard/`

## Purpose of `wizard/`

This directory implements a **multi-step wizard system** for a terminal-based UI (using the INK library). It provides a reusable framework for guiding users through sequential steps (e.g., multi-step forms, interactive assistants) with built-in navigation controls, keyboard shortcuts, and state management.

## Contents Overview

| File | Role |
|------|------|
| `types.ts` | Shared TypeScript type definitions for context value and props |
| `WizardProvider.tsx` | Central provider component; owns all wizard state and navigation logic |
| `useWizard.ts` | Custom hook for child components to consume wizard context |
| `WizardDialogLayout.tsx` | Dialog wrapper that renders the wizard step with a title and navigation footer |
| `WizardNavigationFooter.tsx` | Footer displaying keyboard shortcuts and exit confirmation |
| `index.ts` | Barrel export file exposing the public API |

## How Files Relate to Each Other

```
                              ┌─────────────────────────────────┐
                              │           index.ts              │
                              │    (barrel export - public API) │
                              └──────────────┬──────────────────┘
                                             │ re-exports
                          ┌──────────────────┼──────────────────┐
                          │                  │                  │
                          ▼                  ▼                  ▼
                    WizardProvider    WizardDialogLayout  WizardNavigationFooter
                    WizardContext ◄────── useWizard        useExitOnCtrlCD...
                          │              (hook)                  │
                          │                                       │
                          ▼                                       ▼
                     types.ts                               types.ts
```

**State Ownership**

- `WizardProvider` creates the `WizardContext` and owns all state:
  - `currentStepIndex` — active step position
  - `wizardData` — accumulated data across steps
  - `navigationHistory` — stack for non-linear navigation
  - `isCompleted` — completion flag

**Data Flow**

1. **Initialization**: Consumer wraps app with `<WizardProvider steps={[...]} onComplete={fn} />`
2. **Rendering**: `WizardProvider` renders the current step component (`steps[currentStepIndex]`) inside `WizardContext.Provider`
3. **Consumption**: Child components call `useWizard()` to access state (`currentStepIndex`, `wizardData`) and controls (`goNext`, `goBack`, `goToStep`)
4. **Completion**: When `goNext` reaches the last step, `onComplete(wizardData)` is called via `useEffect`
5. **Exit**: Ctrl+C is wired via `useExitOnCtrlCDWithKeybindings` to interrupt the flow

**Key Files Interaction**

- **`types.ts`** is the single source of truth for types, imported by all other files
- **`WizardDialogLayout`** uses `useWizard` to read step info and renders the step inside a `Dialog`, composing `WizardNavigationFooter` below
- **`WizardNavigationFooter`** shows default keyboard shortcuts (↑↓ navigate, Enter select, Esc go back) and displays "press again to exit" when Ctrl+C is pending

## Key Takeaways

- **Generic typing**: The wizard uses `<T extends Record<string, unknown>>` so callers define their own data shape per step or across the entire wizard
- **Non-linear navigation support**: The `navigationHistory` stack allows jumping to arbitrary steps and returning correctly
- **React Compiler**: Both `WizardProvider` and `WizardDialogLayout` are pre-compiled with React's experimental compiler (visible in the `$` cache variables)
- **Terminal UX**: `WizardNavigationFooter` uses INK components (`Box`, `Text`, `Byline`, `KeyboardShortcutHint`) and provides configurable keyboard shortcuts
- **Separation of concerns**: Provider handles logic/state; layout handles presentation; footer handles shortcuts; the hook provides clean access for any child
