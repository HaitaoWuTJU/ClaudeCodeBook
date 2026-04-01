# Summary of `tools/ReadMcpResourceTool/`

## Purpose of `ReadMcpResourceTool/`

This directory implements a Model Context Protocol (MCP) tool that **reads resources by URI from connected MCP servers**. It provides the core logic, UI rendering, and documentation strings needed to integrate resource reading into a larger tool system.

## Contents Overview

| File | Role |
|------|------|
| `prompt.js` | Exports static string constants (`DESCRIPTION`, `PROMPT`) serving as documentation/templates for the tool system |
| `ReadMcpResourceTool.ts` | Core tool implementation: validates inputs, fetches resources via MCP SDK, handles binary blobs, enforces size limits |
| `UI.tsx` | React/Ink UI rendering: displays "Reading resource X from server Y" messages and pretty-prints JSON results |

## How Files Relate to Each Other

```
prompt.js
   │
   └──► Consumed by the tool registry system
        (provides tool.name → DESCRIPTION/PROMPT mapping)

ReadMcpResourceTool.ts
   │
   ├──► Imports: inputSchema, Output types
   ├──► Imports: UI.renderToolUseMessage, UI.renderToolResultMessage
   └──► Exports: The complete tool definition (inputSchema, outputSchema, call())

UI.tsx
   │
   ├──► Exports: renderToolUseMessage() — called by ReadMcpResourceTool
   ├──► Exports: renderToolResultMessage() — called by ReadMcpResourceTool
   └──► Exports: userFacingName() — returns 'readMcpResource'
```

**Data flow**:
1. Tool system loads `prompt.js` to register tool name/description
2. User invokes `readMcpResource` with `{ server, uri }`
3. `ReadMcpResourceTool.ts` validates server, fetches resource via MCP SDK
4. Binary blobs are decoded from base64, saved to disk, replaced with filepaths
5. `UI.tsx` renders the user-facing messages and formatted JSON output

## Key Takeaways

- **MCP integration**: Communicates with MCP servers via `@modelcontextprotocol/sdk`, sending `resources/read` requests
- **Binary handling**: Automatically detects base64-encoded blobs, decodes them, persists to disk, and returns the filepath
- **Size safety**: Enforces a 100,000-character result limit (`maxResultSizeChars`)
- **Lazy schemas**: Uses deferred Zod schema evaluation for input/output validation
- **Terminal UI**: Rendered using Ink (React for CLI), not browser-based React
- **Read-only operation**: Explicitly marked as non-destructive; only fetches data
