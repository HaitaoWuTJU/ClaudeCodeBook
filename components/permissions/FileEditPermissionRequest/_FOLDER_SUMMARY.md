# Summary of `components/permissions/FileEditPermissionRequest/`

## Purpose of `FileEditPermissionRequest/`

This directory contains the component responsible for presenting a user-facing permission dialog when a file edit operation requires explicit user approval. It displays what changes will be made to a file (as a visual diff) and allows the user to approve, reject, or — via IDE diff support — modify the changes before confirming.

## Contents Overview

| File | Role |
|------|------|
| **`FileEditPermissionRequest.js`** | The main and only component file. It exports `FileEditPermissionRequest`, which serves as the entry point for the permission flow. |

## How Files Relate to Each Other

```
props.input (raw)
        │
        ▼
FileEditPermissionRequest.js
    ├── parses input with FileEditTool.inputSchema (Zod validation)
    ├── builds ideDiffSupport { getConfig, applyChanges }
    ├── extracts display info (basename, relative path)
    ├── renders <FileEditToolDiff> for the visual preview
    ├── renders <FilePermissionDialog> with:
    │       ├── Question text: "Do you want to make this edit to [filename]?"
    │       ├── Subtitle with relative file path
    │       ├── <FileEditToolDiff> diff output
    │       └── Accept / Reject buttons
    └── calls onDone(editedInput) or onReject()
```

The component sits at the boundary between the tool system (`FileEditTool`) and the permission dialog system (`FilePermissionDialog`), bridging them by:

1. **Consuming** raw tool input and validating it through `FileEditTool.inputSchema`
2. **Producing** a structured `FileEditInput` object consumed by `FileEditToolDiff` for rendering
3. **Feeding** the same input into `FilePermissionDialog`, which manages user interaction
4. **Delegating** IDE diff editing to `ideDiffConfig` via the `ideDiffSupport` object

## Key Takeaways

- **Single-file module** — `FileEditPermissionRequest` is self-contained; it imports its dependencies but doesn't export anything else.
- **Zod schema reuse** — Input validation is delegated directly to `FileEditTool.inputSchema`, ensuring consistency between the tool definition and the permission UI.
- **IDE diff integration** — Users can edit diffs in an external IDE; `applyChanges` reconstructs `FileEditInput` from the modified diff, allowing the permission system to act on user-modified edits.
- **React Compiler optimized** — The component uses `_c()` with a 51-slot memoization array to cache rendered elements, avoiding unnecessary re-renders when props haven't changed.
- **Completion type** — Explicitly sets `completionType="str_replace_single"`, signaling to the dialog system that this is a single-file, single-replacement operation.
