# Summary of `components/permissions/NotebookEditPermissionRequest/`

## Purpose of `NotebookEditPermissionRequest/`

This directory provides the permission request UI for **notebook cell editing operations**. When an AI agent wants to modify a Jupyter notebook (insert cells, delete cells, or edit cell contents), this component:

1. **Prompts the user** for permission before allowing the edit
2. **Visualizes the proposed changes** via a diff display
3. **Delegates** the approval/rejection decision to a `FilePermissionDialog`

---

## Contents Overview

| File | Role |
|------|------|
| `NotebookEditPermissionRequest.tsx` | Top-level permission dialog; validates input, builds context, renders `FilePermissionDialog` with diff content |
| `NotebookEditToolDiff.tsx` | Diff visualization sub-component; reads the notebook file, extracts old cell source, computes a patch, and renders `StructuredDiff` or `HighlightedCode` |

---

## How Files Relate to Each Other

```
NotebookEditPermissionRequest.tsx
        │
        │ creates (JSX)
        ▼
┌─────────────────────────────┐
│ FilePermissionDialog         │
│   └─ content: NotebookEditToolDiff  │
└──────────────────────┬──────┘
                       │
                       │ async reads
                       ▼
               NotebookFile (IPynb)
                       │
                       ▼
┌──────────────────────────────────────┐
│ NotebookEditToolDiffInner             │
│   ├─ Extracts old source (by index/id)│
│   ├─ Computes patch hunks             │
│   └─ Renders diff UI                 │
└──────────────────────────────────────┘
```

**Data passed from parent → child:**
| Prop | Type | Purpose |
|------|------|---------|
| `notebook_path` | `string` | Notebook file path |
| `cell_id` | `string \| undefined` | Target cell identifier |
| `new_source` | `string` | Proposed new cell content |
| `cell_type` | `NotebookCellType` | Cell language type |
| `edit_mode` | `"insert" \| "delete" \| "replace"` | Kind of modification |
| `verbose` | `boolean` | Controls diff width and path display |
| `width` | `number` | Terminal width for diff rendering |

---

## Key Takeaways

### Architecture Pattern
The directory follows a **wrapper/inner component pattern**:
- **Outer** (`NotebookEditPermissionRequest`): orchestrates data validation, sets up context, renders the dialog shell
- **Inner** (`NotebookEditToolDiffInner`): handles async I/O and diff computation; consumed via React `use()` inside `Suspense`

### Three Edit Modes
| Mode | Old Source | New Source | Display |
|------|-----------|------------|---------|
| `insert` | — (empty) | ✓ | `HighlightedCode` of new |
| `delete` | ✓ | — (empty) | `HighlightedCode` of old |
| `replace` | ✓ | ✓ | `StructuredDiff` (patch hunks) |

### File I/O Safety
- The notebook file is read **asynchronously** via `getFsImplementation()`
- Reads are **memoized** on `notebook_path` — no redundant disk I/O across renders
- Reads **never throw** — errors return `null`, preventing permission dialog failure

### React Compiler Usage
Both files use the `react/compiler-runtime` (`_c`), enabling:
- Automatic memoization of expensive computations
- Stable references for JSX subtrees via the `$[n]` cache array
- This replaces manual `useMemo`/`useCallback` patterns

### Input Validation
The parent component uses **Zod v4** (`safeParse`) to validate tool input before rendering. On validation failure, it:
1. Logs the error via `logError`
2. Returns a safe default object
3. Prevents the permission dialog from crashing
