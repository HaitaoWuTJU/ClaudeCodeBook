# Summary of `components/permissions/PowerShellPermissionRequest/`

## PowerShellPermissionRequest Directory Summary

### Purpose

The `PowerShellPermissionRequest/` directory implements the user-facing permission workflow for PowerShell commands. It provides interactive dialogs that allow users to approve or deny command execution, optionally configure "always allow" rules with custom prefixes, provide feedback, and understand what permissions are being requested through an explainer panel.

### Contents Overview

| File | Description |
|------|-------------|
| `index.ts` | Re-exports `PowerShellPermissionRequest` component for external consumption |
| `PowerShellPermissionRequest.tsx` | Main React component orchestrating the entire permission dialog UI and logic |
| `powershellToolUseOptions.tsx` | Helper module generating the Yes/No option array with input fields and "always allow" configurations |

### How Files Relate to Each Other

```
index.ts
    └── PowerShellPermissionRequest.tsx (re-export)
            └── powershellToolUseOptions.tsx (imported as helper)
                    └── Provides OptionWithDescription[] → used by Select component
```

**Flow:**

1. **`index.ts`** provides a clean public API entry point, re-exporting the main component.

2. **`PowerShellPermissionRequest.tsx`** is the orchestrator:
   - Reads permission context (`toolUseConfirm`, `toolUseContext`)
   - Uses `usePermissionExplainerUI` and `useShellPermissionFeedback` hooks for UI state
   - Calls `powershellToolUseOptions()` to generate the select options
   - Renders `PermissionDialog` containing `Select`, explainer panel, and debug info
   - Handles user selections → triggers `onAllow` / `onReject` callbacks
   - Tracks analytics events

3. **`powershellToolUseOptions.tsx`** is a pure helper:
   - Accepts suggestions + UI state callbacks
   - Returns typed option objects for the `Select` component
   - Decides whether to show editable prefix input vs. read-only label

### Key Takeaways

1. **Two-layer option system**: Options support both plain button mode (`yes`/`no`) and input field mode (`yes-prefix-edited`/`no`) with user-editable prefix

2. **Editable prefix priority**: The UI prefers an editable prefix input over a read-only suggestions label, falling back only when suggestions contain non-PowerShell rules (directory grants, Read-tool rules)

3. **Windows-specific constraints**: PowerShell intentionally omits sandbox toggle and classifier-reviewed options available in Bash

4. **Keybinding-driven UX**: `Ctrl+E` toggles the explainer, `Ctrl+D` shows debug info, `Tab` switches feedback modes

5. **Analytics integration**: Every user interaction (option selection, explainer visibility, feedback submission) is logged for telemetry

6. **Static prefix extraction**: Compound commands extract prefixes asynchronously, excluding read-only subcommands to avoid redundant "always allow" rules
