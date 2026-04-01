# Summary of `cli/handlers/`

## `cli/handlers/` Directory Summary

### Purpose

This directory serves as the **command dispatch layer** for the Tengu/Claude CLI, containing lazy-loaded handlers for every major `claude <subcommand>` operation. Each file encapsulates the complete lifecycle of a CLI operation—argument parsing, data fetching, business logic, user interaction, and process termination—enabling modular loading so commands only execute when invoked.

### Contents Overview

| File | Primary Responsibilities |
|------|------------------------|
| **auth.ts** | `login`, `status`, `logout` — OAuth flows, token management, API key auth, SSO |
| **agents.ts** | `agents` — Lists all configured agents grouped by source (built-in, user, plugin) |
| **autoMode.ts** | `auto-mode defaults`, `auto-mode config`, `auto-mode critique` — Rule exports and AI review |
| **mcp.ts** | `mcp add`, `mcp remove`, `mcp list`, `mcp get`, `mcp serve` — MCP server lifecycle management |
| **plugins.ts** | `plugin validate`, `plugin list`, `plugin install`, `plugin uninstall`, `plugin marketplace *` — Plugin ecosystem |
| **util.tsx** | `setup-token`, `doctor`, `install` — Setup flows, diagnostics, project initialization |

### How Files Relate to Each Other

```
cli/entry.tsx (main dispatcher)
         │
         ├──────────────────────────────────────────────────────────┐
         │                                                          │
         ▼                                                          ▼
  ┌─────────────────┐                          ┌─────────────────────────┐
  │   Handlers in   │                          │  Shared Service Layer   │
  │   this dir      │                          │  (imported by handlers) │
  ├─────────────────┤                          ├─────────────────────────┤
  │ auth.ts         │──┐                       │ services/analytics/     │
  │ agents.ts       │  │                       │ services/oauth/         │
  │ autoMode.ts     │  │                       │ services/mcp/           │
  │ mcp.ts          │──┤                       │ services/plugins/       │
  │ plugins.ts      │  │                       │ utils/auth.js           │
  │ util.tsx        │──┘                       │ utils/settings/         │
  └─────────────────┘                          │ utils/plugins/          │
                                                │ state/AppState.js       │
                                                └─────────────────────────┘
```

**Shared dependencies across handlers:**

- **`utils/settings/`** — Every handler reads or writes user/project configuration (auto mode, MCP, plugins, agents all have config)
- **`services/analytics/`** — All handlers emit `logEvent()` calls for telemetry (e.g., `tengu_mcp_add_command`, `tengu_doctor_command`)
- **`utils/errors.js`** — `errorMessage()` used universally for formatting error output
- **`utils/log.js`** — `logError()` used for stderr output
- **`utils/slowOperations.js`** — `jsonStringify()` used for structured `--json` output in agents, plugins, and MCP handlers

**Cross-handler patterns:**

- `plugins.ts` and `agents.ts` both surface plugin contributions (plugins can define agents, surfaced in `agents.ts`)
- `mcp.ts` loads plugin MCP servers via `loadPluginMcpServers()` from `utils/plugins/mcpPluginIntegration.js`, bridging plugin and MCP systems
- `util.tsx`'s `DoctorWithPlugins` component uses the same plugin management infrastructure as `plugins.ts`

### Key Takeaways

1. **Lazy loading is intentional design**: Every handler is dynamically imported so `claude help` or commands without subcommands load only the entry point. This keeps cold-start latency low.

2. **Four-layer output strategy**: Handlers produce output via: (a) console output to stdout, (b) `logError()` to stderr, (c) Ink/React terminal UI (interactive flows in auth, mcp, doctor), and (d) `process.exit()` codes for scripting.

3. **Multi-scope configuration is pervasive**: Agents, MCP servers, and plugins all support `local` / `project` / `user` configuration scopes, resolved at read time with a clear priority order.

4. **Analytics is mandatory instrumentation**: Every significant user action (login, plugin operations, MCP commands, doctor runs) emits a `logEvent()` with structured payload and PII-safe field conventions (`_PROTO_*` prefix).

5. **Type safety at boundaries**: Handlers use intra-repo type imports (`ScopedMcpServerConfig`, `AutoModeRules`, `OAuthTokens`, `PluginSource`) to catch configuration contract violations at the dispatch layer.
