# Summary of `components/permissions/FilePermissionDialog/`

## Purpose of `FilePermissionDialog/`

Provides a complete permission dialog system for file operations in a Claude application. It handles the UI and logic for requesting, granting, and rejecting file access permissions (read/write/create), with support for IDE-based diff viewing, symlink detection, session-wide permissions, and contextual feedback input.

## Contents Overview

| File | Purpose |
|------|---------|
| `FilePermissionDialog.tsx` | Main React component—renders the dialog, integrates IDE diff and symlink warnings, orchestrates hooks |
| `permissionOptions.tsx` | Generates permission choice options (accept-once, accept-session, reject) with context-aware labels and descriptions |
| `useFilePermissionDialog.ts` | React hook managing UI state (focus, input modes, feedback), keyboard shortcuts, and option dispatch |
| `usePermissionHandler.ts` | Implementation of permission action handlers—calls analytics, invokes tool callbacks, generates permission update suggestions |
| `ideDiffConfig.ts` | Type definitions and factory function for IDE diff configuration objects |

## How Files Relate to Each Other

```
User selects option in UI
        │
        ▼
FilePermissionDialog.tsx (component)
        │
        ├──► calls useFilePermissionDialog.ts
        │         │
        │         ├──► reads permissionOptions.tsx → options[]
        │         │
        │         ├──► onChange() ──────────────────┐
        │         │                                 │
        │         └──► useKeybindings()            │
        │                                           │
        ▼                                           ▼
usePermissionHandler.ts ◄─────────────────────────┘
        │
        ├──► logs analytics events
        ├──► calls toolUseConfirm.onAllow() / onReject()
        └──► generates permission update suggestions (via filesystem utils)
```

**Data flow summary:**
1. `permissionOptions.tsx` generates the list of available choices (`accept-once`, `accept-session`, `reject`) based on file path, tool permission context, and operation type.
2. `useFilePermissionDialog.ts` wraps the options with interaction logic (focus management, input modes, keyboard shortcuts) and exposes `onChange` for dispatch.
3. When the user selects an option, `useFilePermissionDialog.ts` calls the handler from `usePermissionHandler.ts`.
4. `usePermissionHandler.ts` logs analytics, invokes the original tool callbacks, and generates permission suggestions.
5. `FilePermissionDialog.tsx` composes all of the above into a cohesive UI, adding IDE diff integration (`useDiffInIDE`) and symlink warnings.

## Key Takeaways

- **Generic tool input support**: The system uses `T extends ToolInput` generics and Zod parsing (`parseInput`) to handle different tool input types (file read, file write, notebook edit, etc.) with full type safety.

- **IDE diff integration**: For write operations, the dialog can display changes in the user's IDE instead of inline, delegating the diff UI to `ShowInIDEPrompt`.

- **Symlink awareness**: The component resolves file paths through symlinks (`safeResolvePath`) and warns users when operating on symlinks that point outside the working directory.

- **Special `.claude/` folder handling**: When operating inside project or global `.claude/` folders (for non-read operations), a special "edit own settings" option appears instead of the generic session option.

- **Feedback input modes**: Users can provide optional feedback when accepting or rejecting, with separate input fields toggled via Tab key, all tracked for analytics.

- **Session-level permissions**: `accept-session` options generate path-based or pattern-based permission suggestions via `generateSuggestions()`, allowing Claude to operate on similar files for the remainder of the session.

- **Analytics-first design**: Every permission accept/reject action is logged to both named analytics events and unary completion tracking (`logUnaryEvent`), capturing tool names, language names, feedback presence, and scope information.
