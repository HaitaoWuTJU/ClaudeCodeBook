# Summary of `tools/MCPTool/`

## Purpose of `MCPTool/`

Provides the tool definition and UI rendering infrastructure for MCP (Model Context Protocol) tools in the Claude Code CLI. This module handles classification (search vs. read operations), generic tool template definition, and rich output formatting for MCP tool results.

## Contents Overview

| File | Role |
|------|------|
| `MCPTool.ts` | Generic tool wrapper template; `mcpClient.ts` injects real tool names, descriptions, and execution logic at runtime |
| `prompt.ts` | Placeholder exports (`PROMPT`, `DESCRIPTION`) — actual values come from `mcpClient.ts` |
| `UI.tsx` | Rendering functions for tool inputs, progress updates, and formatted results with intelligent JSON/text unwrapping |
| `classifyForCollapse.ts` | Determines whether MCP tools should collapse in the UI based on allowlisted search/read operations |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                        mcpClient.ts                             │
│  (runtime: injects real tool names, descriptions, call handlers) │
└────────────────────────┬────────────────────────────────────────┘
                         │ Overrides/Provides
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                        MCPTool/                                 │
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────────┐  │
│  │ prompt.ts   │    │ MCPTool.ts   │    │ classifyForCollapse │  │
│  │ (exports)   │───▶│ (template)   │    │ (UI collapse logic) │  │
│  └─────────────┘    └──────┬───────┘    └────────────────────┘  │
│                            │                                     │
│                            ▼                                     │
│                     ┌────────────┐    ┌──────────────────────┐   │
│                     │  UI.tsx    │    │ Tool Registry       │   │
│                     │ (renders)  │    │ (collapses tools)   │   │
│                     └────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

1. **`mcpClient.ts`** instantiates real MCP tools by overriding the placeholder properties in `MCPTool.ts`
2. **`MCPTool.ts`** provides the base tool structure with `call()` being replaced at runtime
3. **`UI.tsx`** consumes tool inputs/outputs to render progress, results, and warnings
4. **`classifyForCollapse.ts`** is queried by the UI layer to decide whether to collapse tool groups (search/read)
5. **`prompt.ts`** exists as a convention file — exports are overridden by `mcpClient.ts` but imported elsewhere to satisfy type/module requirements

## Key Takeaways

- **Pattern**: Generic template + runtime injection — `MCPTool.ts` defines the shape; `mcpClient.ts` fills in the specifics for each installed MCP server
- **Performance**: `classifyForCollapse.ts` uses `Set<string>` for O(1) lookups and normalizes input once, avoiding repeated regex operations
- **Rich output**: `UI.tsx` employs three-tier rendering (unwrap → flatten → fallback) to keep terminal output readable while preserving structure
- **Safety limits**: Hard limits prevent parsing bombs (200K chars for JSON, 100K char results, 12 keys max for flattening)
- **Conservative defaults**: Unknown MCP tools never collapse in the UI, and large responses trigger explicit warnings
- **Feature-gated**: `MCP_RICH_OUTPUT` (via `bun:bundle`) gates advanced rendering strategies
