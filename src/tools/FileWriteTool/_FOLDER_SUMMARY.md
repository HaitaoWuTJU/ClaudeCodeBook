# Summary of `tools/FileWriteTool/`

## Purpose of `FileWriteTool/`

The `FileWriteTool` directory implements a complete tool for writing files to the local filesystem within what appears to be an AI coding assistant (likely Claude Code). It spans the full pipeline from schema definition and input validation through execution and UI rendering.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Main tool definition: schema, validation, execution, hooks |
| `prompt.ts` | Human-readable instructions and descriptions for the AI |
| `UI.tsx` | Terminal UI components for rendering results via Ink |
| `constants.ts` | Shared error/message constants |

## How Files Relate to Each Other

```
User Request
    │
    ▼
┌─────────────────────────────────────────────────┐
│  UI.tsx: renderToolResultMessage()               │
│  (renders create/update/error states in terminal)│
└──────────────────────┬──────────────────────────┘
                       │ reads
┌──────────────────────▼──────────────────────────┐
│  index.ts: call()                               │
│  - validateInput()  ──→  prompt.ts              │
│  - checkPermissions()                           │
│  - expandPath()                                 │
│  - writeTextContent()                           │
│  - LSP notifications                             │
└──────────────────────┬──────────────────────────┘
                       │ produces
┌──────────────────────▼──────────────────────────┐
│  Output schema: { type, filePath, content,      │
│  structuredPatch, originalFile, gitDiff? }      │
└─────────────────────────────────────────────────┘
```

- **`prompt.ts`** defines *what* the AI should do (pre-read requirement, when to use Write vs Edit)
- **`index.ts`** defines *how* to execute it (validation, I/O, permissions, hooks)
- **`UI.tsx`** defines *how* to display results (create confirmations, update diffs, rejections, errors)
- **`constants.ts`** is shared between `index.ts` and `UI.tsx` for error messages

## Key Takeaways

1. **Pre-read enforcement**: The tool will reject writes to existing files if the file hasn't been read through the FileRead tool first (tracked via `readFileState` timestamps).

2. **Dual output paths**: The tool produces distinct `create` vs `update` output types that flow into different UI components (condensed summaries vs full diffs).

3. **Security-first design**: UNC path blocking, team memory secret scanning, and permission system integration happen *before* any I/O.

4. **Line ending discipline**: Unlike previous approaches that sampled repo files (risking `\r` corruption in scripts), this tool preserves the model's explicit line endings and does not rewrite old file endings.

5. **Lazy rejection diffs**: When a write is rejected, the UI uses React `Suspense` to defer the expensive file-read + diff-generation work, avoiding unnecessary I/O for large or missing files.

6. **LSP/VSCode integration**: File writes are synchronized back to language servers and VSCode's diff view in real-time for immediate IDE feedback.

7. **Skill activation**: Writing to a file in a skill directory triggers automatic skill discovery and activation.
