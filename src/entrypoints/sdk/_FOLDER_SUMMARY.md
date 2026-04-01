# Summary of `entrypoints/sdk/`

## Purpose of `entrypoints/sdk/`

This directory provides the **foundation for SDK implementations** that communicate with the Claude Code CLI process. It defines schemas and types for:

1. **Control Protocol**: Bidirectional communication between SDK implementations (Python, Rust, etc.) and the CLI
2. **Data Types**: All serializable types used by the SDK for configuration, messages, hooks, and MCP servers
3. **Public API Surface**: Type-safe interfaces consumed by SDK builders and users

## Contents Overview

| File | Role | Key Contents |
|------|------|---------------|
| **`coreSchemas.ts`** | Schema definitions (source of truth) | Zod v4 schemas for configuration, MCP servers, permissions, hooks (30+ events), and SDK messages |
| **`coreTypes.ts`** | Type re-exports & runtime constants | Generated types from schemas + const arrays (`HOOK_EVENTS`, `EXIT_REASONS`) |
| **`controlSchemas.ts`** | Protocol schemas | Request/response schemas for CLI↔SDK communication (initialization, permissions, MCP management, elicitation) |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SDK Consumer (Python, Rust, etc.)            │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  controlSchemas.ts    │    coreSchemas.ts    │    coreTypes.ts     │
│  ─────────────────    │    ────────────      │    ───────────      │
│  Protocol messages    │    Data schemas      │    Re-exported      │
│  for CLI↔SDK comms    │    (source of truth)  │    generated types  │
│                       │                       │                     │
│  • InitializeRequest  │    • ModelUsage       │    • HOOK_EVENTS[]  │
│  • PermissionRequest │    • McpServerConfig  │    • EXIT_REASONS[] │
│  • McpMessage         │    • PermissionMode   │    • All core types │
│  • Elicitation        │    • HookInput types  │                     │
│  • GetContextUsage    │    • SDKMessage types │                     │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     CLI Process (Node.js)                           │
│  • Reads control requests from SDK (stdin)                           │
│  • Validates with controlSchemas                                     │
│  • Writes responses & SDK messages to stdout                        │
└─────────────────────────────────────────────────────────────────────┘
```

**Data Flow**:

1. **Type Generation Pipeline**:
   - `coreSchemas.ts` (Zod schemas) → `scripts/generate-sdk-types.ts` → `coreTypes.generated.js` → imported by `coreTypes.ts`

2. **SDK Communication**:
   - SDK sends `SDKControlRequestSchema` (via stdin) → CLI validates with `controlSchemas.ts` → processes → sends `SDKControlResponseSchema` or `SDKMessageSchema` (via stdout)

3. **Schema Dependencies**:
   - `controlSchemas.ts` imports `coreSchemas` for types like `McpServerConfig`, `PermissionMode`, `HookEvent`
   - Both schema files use `lazySchema()` wrapper to avoid circular imports

## Key Takeaways

1. **Schema-First Design**: Zod schemas in `coreSchemas.ts` are the source of truth; TypeScript types are auto-generated, not hand-written.

2. **Two-Layer Architecture**:
   - **Core schemas**: Define SDK data structures (messages, config, hooks)
   - **Control schemas**: Define the IPC protocol for SDK↔CLI communication

3. **Lazy Loading Pattern**: All schemas use `lazySchema()` to handle circular dependencies between `coreSchemas.ts` and `controlSchemas.ts`.

4. **Comprehensive Hook System**: 30+ hook event types covering the full SDK lifecycle, allowing SDK consumers to intercept and modify behavior (tool use, permissions, elicitation, file changes, etc.).

5. **MCP Server Flexibility**: Four transport types supported (stdio, SSE, HTTP, SDK) with full lifecycle management (connect, reconnect, toggle, set).

6. **Permission Granularity**: Five destination targets for permission rules, five behavior modes, and decision classification for telemetry.

7. **Protocol Discriminators**: Control requests use `subtype` field; SDK messages use `type` field for discriminated unions.
