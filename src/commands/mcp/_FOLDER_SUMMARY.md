# Summary of `commands/mcp/`

## Purpose of `commands/mcp/`

Provides CLI command handlers for managing **Model Context Protocol (MCP)** servers in Claude Code. The directory implements a layered command architecture: an entry-point loader, a base MCP command with subcommands, a server-add command, and XAA IdP (SEP-990) connection management.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command descriptor with lazy loading — exports metadata and defers to `mcp.tsx` |
| `mcp.tsx` | Command handler — routes `enable`, `disable`, `reconnect`, and default (settings) subcommands |
| `addCommand.ts` | `mcp add <name> <commandOrUrl>` — registers new MCP servers (stdio, SSE, HTTP transports) with OAuth support |
| `xaaIdpCommand.ts` | `mcp xaa setup\|login\|show\|clear` — manages the shared XAA (SEP-990) identity provider configuration |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLI Entry Point                             │
│              "claude mcp [subcommand]"                          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  index.ts  (Command Descriptor)                                  │
│  - type: 'local-jsx'                                            │
│  - immediate: true                                              │
│  - lazy-loads mcp.tsx via import()                              │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  mcp.tsx  (Command Handler / Router)                            │
│  - Routes: enable/disable, reconnect, settings                   │
│  - Renders MCPSettings, MCPReconnect, or PluginSettings         │
│  - Composes with services/mcp/MCPConnectionManager.js            │
│  - Reads state from AppState                                    │
└─────────────────────────────────────────────────────────────────┘
          │                                           │
          ▼                                           ▼
┌─────────────────────┐                 ┌─────────────────────────────┐
│  addCommand.ts       │                 │  xaaIdpCommand.ts           │
│  "mcp add <name>..." │                 │  "mcp xaa setup|login..."   │
│  - Supports stdio,   │                 │  - Manages IdP config       │
│    SSE, HTTP types   │                 │  - Stores secrets in        │
│  - OAuth + XAA       │                 │    keychain                  │
│    support           │                 │  - Shares services with      │
│  - Writes config via │                 │    addCommand (XAA auth)     │
│    addMcpConfig()     │                 │                             │
└─────────────────────┘                 └─────────────────────────────┘
          │                                           │
          ▼                                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              services/mcp/ (shared services)                    │
│  config.js, auth.js, xaaIdpLogin.js, MCPConnectionManager.js     │
└─────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Two command namespaces**: The `mcp` command handles runtime management (enable/disable/reconnect/view settings), while `mcp add` handles server registration and `mcp xaa` handles identity provider configuration.

2. **Three transport types**: MCP servers can be added as local subprocesses (`stdio`), REST APIs (`http`), or Server-Sent Events endpoints (`sse`). Each transport type maps to a distinct configuration schema.

3. **OAuth separation of concerns**: MCP server OAuth credentials are stored alongside server configs, while the shared XAA IdP configuration lives in `userSettings` and uses the system keychain for secrets.

4. **Lazy loading**: `index.ts` uses dynamic `import()` to defer loading the heavy implementation until the command is invoked, keeping the initial CLI startup fast.

5. **XAA is a two-layer concern**: Both `addCommand` (for per-server XAA auth) and `xaaIdpCommand` (for the shared IdP config) import from the same `services/mcp/xaaIdpLogin.js` module, ensuring consistent token/keychain handling.

6. **Protocol enforcement**: The XAA IdP setup enforces HTTPS for all issuers except loopback addresses (`localhost`, `127.0.0.1`, `[::1]`), balancing security with support for local development IdPs.

7. **Analytics tracking**: `addCommand` emits `tengu_mcp_add` events with transport type, scope, and XAA usage flags for product telemetry.
