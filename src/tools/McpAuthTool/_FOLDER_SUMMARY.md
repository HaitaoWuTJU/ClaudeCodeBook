# Summary of `tools/McpAuthTool/`

# `tools/McpAuthTool/`

## Purpose of `McpAuthTool/`

This directory implements a **pseudo-tool pattern** for MCP (Model Context Protocol) servers that require OAuth authentication. Rather than exposing an empty or broken tool set while a server is unauthenticated, this module creates a special "auth trigger" tool that:

1. Accepts no input from the user
2. Initiates the server's OAuth flow
3. Returns an authorization URL for the user to complete in a browser
4. Automatically swaps itself out for the server's real tools, commands, and resources once authentication succeeds

## Contents Overview

| File | Role |
|------|------|
| `McpAuthTool.ts` | Core — defines the tool schema, output types, and the `createMcpAuthTool()` factory function |
| `mcpAuthTool.types.ts` | Shared type definitions for auth output (`McpAuthOutput`) |
| `index.ts` | Public re-export of `McpAuthTool` and its types |

## How Files Relate to Each Other

```
index.ts
  └── exports McpAuthTool.ts and mcpAuthTool.types.ts

McpAuthTool.ts
  ├── imports McpAuthOutput from mcpAuthTool.types.ts
  ├── defines empty Zod inputSchema
  ├── defines getConfigUrl() helper
  ├── defines createMcpAuthTool(serverName, config) → Tool
  │
  └── Tool.execute() orchestrates:
        performMCPOAuthFlow()
          │
          ├─ Promise.race between:
          │     ├─ URL emitter  → user sees auth_url in response
          │     └─ Auth completion → user sees success, no URL
          │
          └─ On success:
                 clearMcpAuthCache()
                 reconnectMcpServerImpl()
                 setAppState() — replaces mcp__<server>__* with real tools
```

The `McpAuthOutput` type flows from `mcpAuthTool.types.ts` into `McpAuthTool.ts` to type the tool's return value.

## Key Takeaways

- **Pattern**: This is a **placeholder tool** — it disappears from `appState.mcp.tools` once real tools are loaded via prefix-based filtering (`mcp__<server>__*`).
- **Transport awareness**: Only `sse` and `http` transports fully support the in-session OAuth flow. The `claudeai-proxy` transport delegates to an external `/mcp` route.
- **Silent auth support**: `Promise.race` elegantly handles cases where the IdP completes auth without needing a browser redirect (e.g., pre-existing SSO session).
- **Side effects on success**: Auth triggers `appState` mutations — it is **not** concurrency-safe (`isConcurrencySafe: false`).
