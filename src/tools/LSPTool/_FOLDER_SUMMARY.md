# Summary of `tools/LSPTool/`

## Purpose of `LSPTool/`

Provides a CLI tool wrapper around the Language Server Protocol (LSP), enabling code intelligence features—go-to-definition, find-references, hover documentation, symbol navigation, and call hierarchy analysis—directly in a terminal environment via the Ink (React for CLI) framework.

## Contents Overview

| File | Role |
|------|------|
| **`prompt.ts`** | Static constants: tool name (`'LSP'`) and a multi-line description listing all 9 supported operations and their parameters |
| **`schemas.ts`** | Zod validation schemas (discriminated union) for tool input; exported type `LSPToolInput` and `isValidLSPOperation()` guard |
| **`symbolContext.ts`** | Extracts the symbol/word at a given file position using regex matching; used to show context in tool use messages |
| **`formatters.ts`** | 14 pure formatting functions transforming LSP response types (Location, DocumentSymbol, Hover, CallHierarchyItem, etc.) into human-readable indented strings |
| **`LSPTool.ts`** | Core orchestration: validates input, waits for LSP server init, opens files, sends LSP requests, filters gitignored locations, calls formatters, and returns structured `{operation, result, resultCount, fileCount}` output |
| **`UI.tsx`** | Ink (React for CLI) components: `renderToolUseMessage()`, `renderToolResultMessage()`, `renderToolUseErrorMessage()`, and `LSPResultSummary` with expandable/verbose views |

## How Files Relate to Each Other

```
prompt.ts (constants: name, description)
         │
         ▼
schemas.ts (validate input → LSPToolInput discriminated union)
         │
         ▼
┌───────────────────────────────────────────────────────────────────────┐
│ LSPTool.ts (orchestrator)                                             │
│   │                                                                   │
│   ├──► symbolContext.ts ──► getSymbolAtPosition() ──► renderToolUseMessage() (UI.tsx) │
│   │                                                                   │
│   ├──► waits for LSP server init                                     │
│   │                                                                   │
│   ├──► opens file in LSP server                                      │
│   │                                                                   │
│   ├──► sends LSP request                                             │
│   │                                                                   │
│   ├──► formatters.ts ──► format*() functions ──► renderToolResultMessage() (UI.tsx)  │
│   │                                                                   │
│   └──► filters gitignored locations                                  │
└───────────────────────────────────────────────────────────────────────┘
```

**Data flow summary**:
1. `schemas.ts` validates raw JSON input → `LSPToolInput` typed discriminated union
2. `LSPTool.ts` coordinates the full pipeline (validation → LSP server communication → formatting → filtering)
3. `symbolContext.ts` resolves symbol names at positions for display context
4. `formatters.ts` converts LSP protocol types into displayable strings
5. `UI.tsx` renders everything using Ink components with expandable views

## Key Takeaways

- **9 LSP operations**: goToDefinition, findReferences, hover, documentSymbol, workspaceSymbol, goToImplementation, prepareCallHierarchy, incomingCalls, outgoingCalls
- **Coordinate convention**: All user-facing inputs are 1-based; internal LSP communication uses 0-based; conversion happens in `LSPTool.ts` via `getMethodAndParams()`
- **Two-step call hierarchy**: `incomingCalls` and `outgoingCalls` first invoke `prepareCallHierarchy` to obtain a `CallHierarchyItem`, then use it in a second request
- **Security measures**: UNC path blocking prevents NTLM credential leaks; `git check-ignore` filters gitignored locations; file size limit (10 MB) prevents memory issues
- **Error resilience**: Falls back gracefully to position-only display when symbol resolution fails; logs warnings for malformed LSP responses
- **React compatibility**: Uses synchronous file I/O (`symbolContext.ts`) because `renderToolUseMessage()` is called from a synchronous React render context
- **Concurrency safe**: Marked as `isConcurrencySafe: true` and `isReadOnly: true`, allowing parallel tool calls
