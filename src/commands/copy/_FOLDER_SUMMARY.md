# Summary of `commands/copy/`

## Purpose of `copy/`

Implements the `/copy` command for the CLI tool, enabling users to copy assistant message content to the clipboard with an interactive picker when multiple code blocks are present.

## Contents Overview

The directory contains two files forming a **lazy-loading pattern**:

| File | Role | Lines | Description |
|------|------|-------|-------------|
| `index.ts` | Entry point | ~25 | Lightweight metadata + dynamic import for code splitting |
| `copy.tsx` | Implementation | ~300 | Full React component with clipboard logic, markdown parsing, and interactive picker |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│  index.ts (entry point)                                  │
│  - Exports copy command metadata                         │
│  - load() returns dynamic import('./copy.js')           │
└────────────────────────┬────────────────────────────────┘
                         │ lazy-loaded on first use
                         ▼
┌─────────────────────────────────────────────────────────┐
│  copy.tsx (implementation)                              │
│  - Extracts assistant messages from context             │
│  - Parses markdown with marked                          │
│  - Copies via OSC 52 + temp file fallback               │
│  - Renders CopyPicker React component                    │
└─────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Lazy Loading**: The `index.ts` entry point defers loading the entire copy implementation (including `marked`, React, clipboard logic) until the user actually invokes `/copy`

2. **Markdown Code Extraction**: The implementation parses assistant messages to identify code blocks, allowing users to copy either the full response or specific individual code blocks

3. **Dual Clipboard Strategy**: Uses the terminal's OSC 52 clipboard protocol for direct clipboard access, while also writing to a temp file (`$TMPDIR/claude/`) as a reliable fallback

4. **Interactive Picker**: When multiple code blocks exist, renders a keyboard-driven `CopyPicker` component with options for full response or specific blocks

5. **Analytics Integration**: Logs copy events with metadata (block count, message age, selected block) for usage tracking

6. **Safety Measures**: Includes path traversal protection when determining file extensions from language identifiers

7. **Config Persistence**: Supports an "always copy full" preference that saves to global config for future sessions
