# Summary of `components/permissions/ExitPlanModePermissionRequest/`

# ExitPlanModePermissionRequest.tsx

## Purpose

`ExitPlanModePermissionRequest.tsx` renders a permission request dialog that appears when a user attempts to exit plan mode in Claude Code. The component manages the complex workflow of:

- Displaying plan content (V1/V2 plans) for user review
- Allowing external editor integration via `Ctrl+G`
- Providing multiple approval/rejection options (bypass permissions, accept edits, auto mode, Ultraplan)
- Auto-naming sessions from plan content
- Handling image paste events during plan review
- Tracking analytics for plan exit events

## Contents Overview

### Imports
The file imports React hooks and context (`useState`, `useRef`, `useCallback`), app state management, notification utilities, plan reading utilities, image handling, analytics, and various permission-related utilities.

### Key Functions

| Function | Purpose |
|----------|---------|
| `buildPermissionUpdates()` | Constructs permission update payloads for plan approval, including prompt-based rules when classifier permissions are enabled |
| `autoNameSessionFromPlan()` | Extracts a session name from plan content (first 1000 chars) and saves it to session storage on acceptance |
| `buildPlanApprovalOptions()` | Generates the array of selectable response options based on current mode, context usage, and feature flags |
| `getContextUsedPercent()` | Calculates context window usage percentage for display |
| `handleResponse()` | Core handler that processes user selections, managing V1/V2 mode differences, Ultraplan integration, auto mode transitions, and permission updates |
| `onImagePaste()` | Handles clipboard image paste events, caching and storing images for plan attachment |
| `handleKeyDown()` | Manages keyboard shortcuts: `Ctrl+G` for external editor, `Shift+Tab` for auto-accept |

### Main Component: `ExitPlanModePermissionRequest`

Implements the `PermissionRequestProps` interface and renders a `PermissionDialog` containing:

- **Header**: Shows plan mode context with appropriate messaging
- **Plan Display**: Renders plan content via `Markdown` component (V1) or direct text (V2)
- **Options**: Renders `Select` component with available approval options
- **Actions**: Submit button, cancel button, and Ultraplan button (when `ULTRAPLAN` flag is enabled)
- **Image Pasting**: Overlay handler for clipboard image capture
- **Keyboard Handling**: Document-level listener for `Ctrl+G` and `Shift+Tab`

### Response Values

| Value | Meaning |
|-------|---------|
| `yes-bypass-permissions` | Accept plan, bypass all permission checks |
| `yes-accept-edits` | Accept plan, apply edits with permission checks |
| `yes-accept-edits-keep-context` | Accept edits, keep full context |
| `yes-default-keep-context` | Default approval, keep context |
| `yes-resume-auto-mode` | Resume auto mode after exit |
| `yes-auto-clear-context` | Accept, automatically clear context when needed |
| `ultraplan` | Launch Ultraplan assistant |
| `no` | Reject/cancel |

## Key Takeaways

1. **V1/V2 Mode Handling**: The component differentiates between V1 (inline plan via `input.plan`) and V2 (file-based plan via `getPlan()`) by checking the tool name (`EXIT_PLAN_MODE_V2_TOOL_NAME`), not the plan content itself, due to PR #10394 which changed how plans are injected.

2. **Feature-Gated Options**: Approval options are dynamically built based on feature flags (`ULTRAPLAN`, `TRANSCRIPT_CLASSIFIER`), current mode, and context usage percentage.

3. **External Editor Integration**: Full `$EDITOR` support via `editPromptInEditor()` (V1) or `editFileInEditor()` (V2), triggered by `Ctrl+G`.

4. **Analytics Tracking**: Uses `getPewterLedgerVariant()` to track plan exit events with metadata including plan length, outcome, and interview phase.

5. **Fire-and-Forget Auto-Naming**: `autoNameSessionFromPlan()` is called asynchronously without awaiting, ensuring it doesn't block the main flow.

6. **Image Handling**: Supports image paste during plan review by storing images and creating a pending notification for attachment handling.

7. **Keyboard Shortcuts**: `Shift+Tab` immediately selects the auto-accept option, bypassing the normal UI flow for power users.
