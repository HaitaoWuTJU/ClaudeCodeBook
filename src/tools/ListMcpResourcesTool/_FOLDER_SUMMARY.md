# Summary of `tools/ListMcpResourcesTool/`

## Purpose of `ListMcpResourcesTool/`

This directory implements a single tool, `ListMcpResourcesTool`, that queries connected MCP (Model Context Protocol) servers to list their available resources. The tool supports optional filtering by server name and handles caching, concurrency, and graceful error handling for multi-server scenarios.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | **Core implementation**: Tool definition, input/output schemas, `call()` logic, result transformation |
| `prompt.ts` | **Static configuration**: Tool name constant, description text, and user-facing prompt |
| `UI.tsx` | **Terminal rendering**: Ink/React components for displaying tool use messages and results |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                        prompt.ts                             │
│  • Exports: LIST_MCP_RESOURCES_TOOL_NAME                     │
│  • Exports: DESCRIPTION (usage examples)                     │
│  • Exports: PROMPT (parameter documentation)                 │
└─────────────────────────┬───────────────────────────────────┘
                          │ (consumed by)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                         index.ts                             │
│  • Imports name, description, prompt from prompt.ts          │
│  • Defines Zod schemas (input → optional server filter)      │
│  • Builds tool via buildTool()                               │
│  • Implements call() → fetches from MCP clients              │
│  • Exports: ListMcpResourcesTool, Output type                │
└──────────────┬──────────────────────┬────────────────────────┘
               │                      │
               ▼                      ▼
┌──────────────────────────┐  ┌────────────────────────────────┐
│           UI.tsx         │  │        Other consumers          │
│  • Imports Output type   │  │  • MCP tool registration        │
│  • Exports render*() fns │  │  • CLI/UI rendering             │
└──────────────────────────┘  └────────────────────────────────┘
```

## Key Takeaways

1. **Caching Strategy**: The tool uses LRU-cached `fetchResourcesForClient` with invalidation on connection close or `resources/list_changed` notifications, ensuring fresh results without redundant fetches.

2. **Graceful Degradation**: When querying multiple servers, one server's failure returns `[]` for that server instead of sinking the entire operation.

3. **Result Size Management**: Output is capped at 100,000 characters to prevent terminal overflow, with truncation detection available.

4. **Read-Only & Safe**: The tool is marked as `isReadOnly: true` and `isConcurrencySafe: true`, with `shouldDefer: true` to avoid blocking.

5. **User-Friendly Empty State**: When no resources are found, users receive an encouraging message explaining that MCP servers may still provide tools even without resources.

6. **UI Separation**: Terminal rendering is cleanly separated into `UI.tsx` using Ink (React for CLIs), keeping the core logic decoupled from presentation.
