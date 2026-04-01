# Summary of `components/permissions/FileWritePermissionRequest/`

## Purpose of `FileWritePermissionRequest/`

This directory contains the React components responsible for rendering a **permission request dialog** when a tool attempts to write to a file. The components display the proposed changes (old vs. new content) and allow users to approve, reject, or modify the write operation before it executes.

## Contents Overview

| File | Description |
|------|-------------|
| `FileWritePermissionRequest.tsx` | **Main entry point** — orchestrates the permission dialog, reads file content, determines if file exists, provides IDE diff support for editing, and renders the `FilePermissionDialog` with diff sub-components |
| `FileWriteToolDiff.tsx` | **Diff visualization** — renders a visual diff between `oldContent` and `content`, showing either a structured multi-line diff (via `StructuredDiff`) or syntax-highlighted code (via `HighlightedCode`) depending on file state |

## How Files Relate to Each Other

```
FileWritePermissionRequest.tsx (parent orchestrator)
│
├── reads file at file_path → gets oldContent
├── parses input → gets content (new)
├── creates ideDiffSupport config
└── renders FilePermissionDialog
    │
    └── FileWriteToolDiff.tsx (child renderer)
        │
        ├── receives: file_path, content, fileExists, oldContent
        ├── computes: hunks via getPatchForDisplay()
        └── outputs: StructuredDiff (existing file with changes)
                     or HighlightedCode (new file)
```

**Data flow:** `FileWritePermissionRequest.tsx` is the container that owns the permission logic and state. It reads the filesystem, parses tool input, and delegates the actual rendering of content changes to `FileWriteToolDiff.tsx`, which focuses purely on the visual presentation of the diff.

## Key Takeaways

- **Single read pattern**: The parent component reads `oldContent` once and passes it to both the permission text ("Create" vs. "Overwrite") and the diff renderer—avoiding redundant `existsSync` checks on slow filesystems.

- **IDE diff editing**: The `ideDiffSupport` configuration in the parent enables users to modify proposed content in an IDE-style diff view before the write executes; changes are applied via `applyChanges` → `firstEdit.new_string`.

- **React compiler optimization**: Both files use the `_c` (react/compiler-runtime) with slot-based cache arrays (`$` and `_$`) for manual memoization of computed values and JSX subtrees.

- **Conditional rendering in diff**: `FileWriteToolDiff` returns `null` for new files, a structured diff for files with changes, or highlighted code when no hunks exist—handling all three states cleanly.

- **Terminal-aware**: `FileWriteToolDiff` uses `useTerminalSize` to compute `columns`, passing `columns - 2` to `StructuredDiff` to account for box borders.
