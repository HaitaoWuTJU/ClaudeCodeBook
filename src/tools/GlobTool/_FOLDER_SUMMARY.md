# Summary of `tools/GlobTool/`

## Purpose of `GlobTool/`

The `GlobTool/` directory implements a **filesystem pattern-matching search tool** as part of a larger MCP (Model Context Protocol) server codebase. It enables users to find files by glob/wildcard patterns (e.g., `**/*.js`, `src/**/*.ts`) within a specified directory. The tool is designed for scalability across any codebase size and returns matching file paths sorted by modification time.

---

## Contents Overview

| File | Role | Lines |
|------|------|-------|
| `index.ts` | **Entry point** — re-exports all public members (tool, schemas, UI helpers) | ~20 |
| `GlobTool.ts` | **Core implementation** — input validation, permission checks, glob execution, result formatting | ~200 |
| `UI.tsx` | **User-facing rendering** — display names, usage messages, error messages, result summaries | ~90 |
| `prompt.ts` | **Metadata constants** — tool name (`"Glob"`) and usage description | ~10 |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                      index.ts                           │
│  (Re-exports: GlobTool, inputSchema, outputSchema,      │
│   renderToolUseMessage, userFacingName, etc.)           │
└─────────────────────┬───────────────────────────────────┘
                      │ imports
          ┌───────────┼────────────┐
          ▼           ▼            ▼
┌─────────────────┐  ┌──────────┐  ┌──────────────────┐
│  GlobTool.ts    │  │  UI.tsx  │  │   prompt.ts      │
│                 │  │          │  │                  │
│ - Validates     │  │ - Renders │  │ - GLOB_TOOL_NAME │
│   input/paths   │  │   user    │  │ - DESCRIPTION    │
│ - Checks perms   │  │   messages│  │                  │
│ - Runs glob()   │  │ - Formats │  └──────────────────┘
│ - Formats output│  │   errors  │
└────────┬────────┘  └────┬─────┘
         │                │
         │                │ imports (re-export)
         └────────────────┴──→ index.ts re-exports UI helpers
```

**Data flow through the tool lifecycle:**

1. **Registration** — `index.ts` exports `GlobTool` (built via `buildTool()` with input/output schemas from `GlobTool.ts`) to the tool registry.
2. **Invocation** — When called, `GlobTool.call()` validates the `pattern` (required) and `path` (optional), checks filesystem permissions, runs the glob search, and returns matched files.
3. **Display** — `UI.tsx` functions format invocation details and results for CLI output, with errors handled contextually (file not found vs. generic errors).

---

## Key Takeaways

- **Security first**: The tool explicitly rejects UNC paths (`\\` or `//`) to prevent NTLM credential leakage, and enforces read-permission checks before execution.
- **Scalable by design**: Results are relativized under `cwd` to minimize token usage, and capped at 100 matches by default with a truncation flag.
- **Read-only & concurrency-safe**: Marked with `isReadOnly: true` and `isConcurrencySafe: true` for safe parallel execution.
- **DRY via reuse**: `renderToolResultMessage` is borrowed directly from `GrepTool`, indicating shared result-display logic across search tools in the codebase.
- **Schema-validated I/O**: All inputs and outputs are validated via Zod v4 schemas, ensuring type safety and preventing malformed tool calls.
- **CLI-optimized rendering**: The UI layer distinguishes between verbose (full paths, raw errors) and non-verbose (abbreviated paths, user-friendly messages) display modes.
