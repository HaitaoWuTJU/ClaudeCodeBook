# Summary of `tools/FileEditTool/`

## Purpose of `FileEditTool/`

Implements the **Edit Tool** (`FileEditTool`) for a Claude Code-like application, enabling precise in-place file modifications using exact string replacement. The tool validates permissions, enforces read-before-write patterns, performs atomic write operations with backup support, and integrates with LSP servers and VSCode for editor synchronization.

---

## Contents Overview

| File | Role |
|------|------|
| [`constants.ts`](./constants.ts) | Defines shared constants: tool name, permission patterns, error messages |
| [`FileEditTool.ts`](./FileEditTool.ts) | **Core implementation**: input validation, permission checks, atomic write execution, LSP/VSCode notifications, analytics logging |
| [`prompt.ts`](./prompt.ts) | **User documentation**: generates usage instructions and rules for file editing |
| [`types.ts`](./types.ts) | **Schema definitions**: Zod schemas for tool input/output validation using lazy evaluation to avoid circular dependencies |
| [`UI.tsx`](./UI.tsx) | **Presentation layer**: React components for rendering success messages, rejections with diffs, and error states |
| [`utils.ts`](./utils.ts) | **Helpers**: applies edits, generates diff patches, normalizes quote styles, creates context snippets |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              USER INPUT                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  types.ts ──────► Schema validation (Zod)                                │
│                   Input: { file_path, old_string, new_string, edits? }  │
│                   Output: FileEditOutput with structuredPatch           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  prompt.ts                                                               │
│  Provides user-facing instructions and rules for the tool               │
│  References FILE_READ_TOOL_NAME (imported from FileReadTool)           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  constants.ts                                                            │
│  Exports: FILE_EDIT_TOOL_NAME, permission patterns,                     │
│           FILE_UNEXPECTEDLY_MODIFIED_ERROR                              │
│  Used by: FileEditTool.ts, types.ts                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  FileEditTool.ts (CORE)                                                  │
│  1. ValidateInput: Checks permissions (wildcard matching),              │
│                    file existence, string uniqueness, size limits      │
│  2. Skill discovery (if not CLAUDE_CODE_SIMPLE)                         │
│  3. Atomic write: backup → validate → apply → notify                    │
│  4. LSP: didChange + didSave                                             │
│  5. VSCode SDK notification                                              │
│  6. Analytics logging                                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  utils.ts (used by both Core and UI)                                    │
│  • normalizeFileEditInput() ──► prepare edit with quote handling       │
│  • getPatchForEdits() ───────► apply edits and generate diff           │
│  • preserveQuoteStyle() ─────► maintain curly/straight quotes          │
│  • getSnippet() ─────────────► generate context snippets                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  UI.tsx                                                                  │
│  • renderToolUseMessage() ────► shows file being edited                 │
│  • renderToolResultMessage() ► shows success with patch diff           │
│  • renderToolUseRejected() ──► shows rejection with computed diff       │
│  • EditRejectionDiff ─────────► Suspense-wrapped async diff loader     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              USER OUTPUT                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

**Dependency Direction**: `types.ts` → `constants.ts` → `FileEditTool.ts` ↔ `utils.ts` → `UI.tsx`. The `prompt.ts` is consumed by the tool framework independently.

---

## Key Takeaways

### Architecture

- **Separation of concerns**: Input validation (`types.ts`), core logic (`FileEditTool.ts`), utilities (`utils.ts`), and presentation (`UI.tsx`) are cleanly separated
- **Lazy schema evaluation**: Uses `lazySchema()` wrapper in `types.ts` to break circular TypeScript dependency chains
- **Self-contained file**: Explicit comment in `constants.ts` notes it exists to avoid circular dependencies

### Security

- **UNC path blocking**: Explicitly rejects `\\` and `//` paths before any filesystem operation
- **Permission patterns**: Respects tool permission deny rules with wildcard matching (`/.claude/**`, `~/.claude/**`)
- **Team memory protection**: Prevents introducing secrets into team memory files

### Reliability

- **Atomic write with backup**: Creates history backup before modification, with double-checked modification timestamps
- **Read-before-write enforcement**: Tool errors unless file has been previously read in the session
- **Race condition handling**: Validates no external modifications between read and write

### User Experience

- **Quote style preservation**: Automatically maintains curly/straight quotes from matched text in new text
- **Interactive diff feedback**: Shows structured diffs in rejections and results via `UI.tsx`
- **Context snippets**: Displays relevant code context around edits (4–8 lines) with line numbers
- **LSP integration**: Notifies language servers of changes for real-time diagnostics

### Performance

- **Chunked context reads**: Diff computation reads bounded windows (`CONTEXT_LINES`) around edit location, not entire files
- **Snippet truncation**: Attachment snippets capped at 8KB to prevent excessive output
- **Batched edits**: `replace_all` parameter supports bulk replacements in single operation
- **Skip intermediate patch**: `utils.ts` directly calls `getPatchFromContents` for ~20% improvement on large files
