# Summary of `tools/NotebookEditTool/`

## Purpose of `NotebookEditTool/`

Implements a CLI tool that enables editing of Jupyter notebook cells (.ipynb files) with support for three edit modes: replace, insert, and delete. The module follows a modular architecture separating constants, prompts, core logic, and UI rendering into distinct files to avoid circular dependencies and maintain separation of concerns.

## Contents Overview

| File | Role |
|------|------|
| `constants.ts` | Single string export (`'NotebookEdit'`) — isolated to break circular import chains |
| `prompt.ts` | Instructional strings (`DESCRIPTION`, `PROMPT`) defining tool usage rules, indexing, and file requirements |
| `NotebookEditTool.ts` | Core implementation: Zod schemas, validation logic, permission checks, file I/O, cell manipulation, state updates, and attribution tracking |
| `UI.tsx` | React/Ink UI components for rendering tool invocations, rejections, errors, and results in the terminal |

## How Files Relate to Each Other

```
constants.ts          prompt.ts
       │                    │
       └────────┬───────────┘
                ▼
       ┌────────────────┐
       │NotebookEditTool│ ◄──── Core orchestrator
       │    .ts         │
       └───────┬────────┘
               │
               ├────────────────┐
               ▼                ▼
        UI.tsx ─────────► Tool consumers
    (renders results)    (use tool)
```

**Import chain:**
1. `constants.ts` → `NotebookEditTool.ts` (provides tool name)
2. `prompt.ts` → `NotebookEditTool.ts` (provides input/output schemas via `DESCRIPTION`/`PROMPT`)
3. `NotebookEditTool.ts` → `UI.tsx` (exports `Output` type and `inputSchema` consumed by UI components)
4. `UI.tsx` → `NotebookEditTool.ts` (renders results using exported schemas and types)

**Data flow during execution:**
```
User input → NotebookEditTool.ts:validateInput()
         → NotebookEditTool.ts:checkPermissions()
         → File read (with metadata preservation)
         → JSON parse (non-memoized to protect shared cache)
         → Cell operation (replace/insert/delete)
         → File write (restore original encoding/line endings)
         → State updates (readFileState, fileHistory)
         → Output → UI.tsx:renderToolResultMessage()
```

## Key Takeaways

- **Read-before-edit enforcement**: The tool requires notebooks to have been read in the current context and detects external modifications via mtime, preventing stale data overwrites (error codes 9–10).

- **Isolation pattern**: `constants.ts` exists solely to prevent circular dependencies—a documented anti-pattern used throughout the codebase.

- **Notebook-aware operations**:
  - Generates 13-character random IDs for `nbformat >= 4.5` notebooks
  - Clears execution state (`execution_count`, `outputs`) when replacing code cells
  - Resolves cell references by UUID or `cell-N` numeric index format

- **UNC path blocking**: Filesystem writes are intentionally skipped for UNC paths (`\\`, `//`) to prevent NTLM credential leakage.

- **Two-mode UI**: All `UI.tsx` components support a `verbose` toggle—concise mode shows minimal info (file path, cell ID); verbose mode reveals full content previews, cell types, and edit modes.

- **Output attribution**: The tool returns `original_file` and `updated_file` fields to support downstream attribution and audit logging of file modifications.
