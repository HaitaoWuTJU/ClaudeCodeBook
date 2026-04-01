# Summary of `commands/export/`

## Purpose of `export/`

This directory implements the **`/export` command** for a CLI tool, enabling users to export conversation messages to plain text files. The implementation supports two modes: writing directly to a file (when a filename argument is provided) or displaying an interactive dialog for guided export.

## Contents Overview

| File | Role | Description |
|------|------|-------------|
| `index.ts` | Command definition | Entry point that registers the command with metadata and lazy-loads the implementation |
| `export.js` | Implementation | Core logic: renders messages to text, handles filename generation, manages direct write vs dialog flow |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│  Command Registry (imports index.ts)                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │ loads command definition
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  index.ts (Command Definition)                                  │
│  ├── name: 'export'                                            │
│  ├── type: 'local-jsx'                                         │
│  ├── load: () => import('./export.js')  ← Lazy loading         │
│  └── argumentHint: '[filename]'                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │ loaded on demand
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  export.js (Implementation)                                    │
│  ├── formatTimestamp()     → Date formatting                   │
│  ├── extractFirstPrompt()  → Message content extraction        │
│  ├── sanitizeFilename()    → Filename sanitization             │
│  ├── exportWithReactRenderer() → React→plaintext conversion    │
│  └── call()                → Main entry: direct write or dialog │
└─────────────────────────────────────────────────────────────────┘
```

The **lazy loading pattern** ensures that the React rendering utilities and heavy dependencies are only loaded when the export command is actually invoked, keeping the command registry lightweight.

## Key Takeaways

1. **Two-mode export system**:
   - **Direct write**: User provides filename via `/export myfile.txt`
   - **Dialog mode**: Interactive React component when no argument provided

2. **Automatic filename generation**: If no filename is given, creates `{timestamp}-{prompt-preview}.txt` or `conversation-{timestamp}.txt` from the conversation's first user message

3. **Lazy loading architecture**: Implementation is separated from the command definition for faster CLI startup

4. **Content extraction**: Handles complex message structures (string or content block arrays) and extracts first text lines for filenames

5. **React integration**: Uses `renderMessagesToPlainText()` to convert React message components into plain text output
