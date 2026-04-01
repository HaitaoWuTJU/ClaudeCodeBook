# Summary of `components/permissions/rules/`

## Purpose of `rules/`

The `rules/` directory contains **UI components for managing permission rules** in a CLI application (Claude Code). These components handle user-facing dialogs, tabs, and inputs for viewing, adding, and removing permission configurations—particularly workspace directory access and tool execution permissions.

---

## Contents Overview

| File | Purpose | User Action |
|------|---------|-------------|
| `AddPermissionRules.tsx` | Dialog to add permission rules with destination selection | User selects where to save: local/project/user settings |
| `AddWorkspaceDirectory.tsx` | Input dialog with autocomplete for adding directories | User types/picks a directory path, optionally remembers it |
| `RemoveWorkspaceDirectory.tsx` | Confirmation dialog for removing a directory | User confirms "Yes/No" to revoke access |
| `PermissionRuleDescription.tsx` | Displays human-readable text for a permission rule | Renders rule summaries (e.g., "Any Bash command starting with...") |
| `RecentDenialsTab.tsx` | Tab listing auto-mode classifier denials | User approves or marks denials for retry |
| `WorkspaceTab.tsx` | Main tab listing current workspace directories | User selects "Add directory..." or removes existing dirs |

---

## How Files Relate to Each Other

```
WorkspaceTab.tsx (main tab)
├── AddWorkspaceDirectory.tsx (add directory flow)
│   ├── PermissionRuleDescription.tsx (shows "Claude Code can read/edit files in [path]")
│   └── DirectoryInput with autocomplete
├── RemoveWorkspaceDirectory.tsx (remove directory flow)
│   └── Confirmation dialog → updates permissionContext
├── AddPermissionRules.tsx (add rules flow)
│   ├── Used for other rule types beyond directories
│   └── PermissionRuleDescription.tsx (rule descriptions)
└── RecentDenialsTab.tsx (audit denials)
    └── approve/retry interactions
```

**Shared infrastructure:**
- All use **Ink.js** (`Box`, `Text`) for terminal rendering
- All integrate with **`ToolPermissionContext`** via `setToolPermissionContext`
- **Permission updates** flow through `applyPermissionUpdate()` → `persistPermissionUpdate()`
- **Unreachable rule detection** via `detectUnreachableRules()` in `AddPermissionRules`
- All use the **custom `Select`** component for user choices

---

## Key Takeaways

1. **Full CRUD for permissions**: The directory implements Create (Add), Read (PermissionRuleDescription), and Delete (Remove) operations for permission rules.

2. **Ink-based terminal UI**: Unlike web React, these components render to the terminal using Ink—a React-compatible framework for CLI tools.

3. **React Compiler adoption**: All files use `react/compiler-runtime` with manual `$` cache arrays, indicating active migration toward React Compiler optimization.

4. **Permission scoping**: Rules can be saved to different destinations (`localSettings`, `projectSettings`, `userSettings`), enabling fine-grained control over where permissions persist.

5. **Unreachable rule detection**: When adding rules, the system detects and reports shadowed rules that would never be reached—preventing user confusion.

6. **Auto-complete for directories**: The directory input leverages `getDirectoryCompletions()` for UX-friendly path entry with keyboard navigation (Tab, ↑/↓, Ctrl+P/Ctrl+N).

7. **Memory options**: Users can choose to remember directory access "for this session only" or persistently, with options stored in `RememberDirectoryOption` (`yes-session`, `yes-remember`).

8. **Error styling for destructive actions**: Removal dialogs use `color="error"` to visually warn users before revoking access.

9. **Tab-based navigation**: The `WorkspaceTab` coordinates focus between the tab header and its child `Select`, properly delegating keyboard navigation (`onUpFromFirstItem={focusHeader}`).

10. **Compiled output**: Several files appear as Babel-transformed React Compiler output with embedded source maps (`sourceMappingURL`), suggesting a build pipeline that pre-compiles components.
