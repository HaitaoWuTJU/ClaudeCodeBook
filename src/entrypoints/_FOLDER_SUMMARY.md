# Summary of `entrypoints/`

## Purpose of `entrypoints/`

The `entrypoints/` directory serves as the **public API surface** for Claude Code, exposing multiple distinct runtime targets for different use cases:

| Entrypoint | Role |
|------------|------|
| `cli.tsx` | Primary CLI bootstrap — routes all `claude` commands and subcommands |
| `init.ts` | Runtime initialization — sets up config, network, telemetry, OAuth |
| `mcp.ts` | MCP server via STDIO — exposes tools to external MCP clients |
| `agentSdkTypes.ts` | Agent SDK types re-export — public types for SDK builders |
| `sandboxTypes.ts` | Sandbox configuration types — Zod schemas for sandbox settings |
| `sdk/` | SDK foundation — schemas, types, and control protocol for SDK implementations |

## Contents Overview

### Top-Level Entrypoints

| File | Language | Lines | Purpose |
|------|----------|-------|---------|
| `cli.tsx` | TSX | ~450 | Bootstrap CLI with fast-path routing for flags/subcommands |
| `init.ts` | TS | ~280 | Runtime initialization (config, network, telemetry, OAuth) |
| `mcp.ts` | TS | ~300 | MCP STDIO server exposing tools to external clients |
| `agentSdkTypes.ts` | TS | ~600 | Agent SDK public types and stub functions |
| `sandboxTypes.ts` | TS | ~200 | Zod schemas for sandbox configuration |

### SDK Subdirectory (`sdk/`)

| File | Purpose |
|------|---------|
| `coreSchemas.ts` | Zod v4 schemas for all serializable SDK data types |
| `coreTypes.ts` | Re-exports generated types + const arrays (`HOOK_EVENTS`, `EXIT_REASONS`) |
| `controlSchemas.ts` | Control protocol schemas for CLI↔SDK IPC communication |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CLI Bootstrap (cli.tsx)                           │
│  • Parses process.argv, routes to fast-paths or loads ../main.js            │
│  • Calls init() before full CLI to set up runtime environment               │
└─────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Initialization (init.ts)                                 │
│  • Validates config, configures network/proxy/mTLS                          │
│  • Sets up telemetry (deferred), OAuth, remote settings                      │
│  • Calls profileCheckpoint() for startup profiling                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
┌──────────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐
│  MCP Server (mcp.ts)      │  │  Agent SDK (agentSdk) │  │  CLI Main        │
│  • STDIO transport        │  │  • Re-exports types   │  │  (main.js)       │
│  • getTools() + exec      │  │  • Session management │  │                  │
│  • LRU file state cache   │  │  • Query/call APIs    │  │                  │
│  • JSON Schema output     │  │  • Daemon primitives  │  │                  │
└──────────────────────────┘  │  • MCP server factory │  │                  │
                               └──────────┬───────────┘  └──────────────────┘
                                          │
                                          ▼
                    ┌───────────────────────────────────────────────────────┐
                    │            SDK Foundation (sdk/)                       │
                    │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
                    │  │coreSchemas  │  │controlSchemas│  │coreTypes     │  │
                    │  │• Zod schemas│  │• IPC protocol│  │• Re-exports  │  │
                    │  │• 30+ hooks  │  │• Requests    │  │• Generated   │  │
                    │  │• MCP config │  │• Responses   │  │• Constants   │  │
                    │  └─────────────┘  └──────────────┘  └──────────────┘  │
                    └───────────────────────┬─────────────────────────────────┘
                                            │
                                            ▼
                    ┌───────────────────────────────────────────────────────┐
                    │     SDK Implementations (Python, Rust, etc.)           │
                    │  • Communicate via control protocol (stdin/stdout)   │
                    │  • Consume coreTypes + coreSchemas                    │
                    └───────────────────────────────────────────────────────┘
```

**Dependency Chain**:

1. `cli.tsx` → `init.ts` (runtime setup) → `cli/main.js` (full CLI)
2. `cli.tsx` → `mcp.ts` (MCP server mode) → `tools.js`, `Tool.js` (tool definitions)
3. `cli.tsx` → `agentSdkTypes.ts` (SDK mode) → `sdk/` (types and schemas)
4. `agentSdkTypes.ts` → `sandboxTypes.ts` (sandbox config types)
5. `sdk/` internal: `coreSchemas.ts` → type generation → `coreTypes.ts`

## Key Takeaways

### 1. Multiple Runtime Modes via Single Entrypoint

`cli.tsx` acts as a **router**, parsing arguments at the top level and dispatching to specialized entrypoints without loading their modules upfront. This enables:

- Instant `--version` output (zero imports)
- Feature-gated code elimination via `bun:bundle`
- Minimal module evaluation for fast-path commands

### 2. Initialization is Separate from Bootstrap

`init.ts` runs **before** the full CLI to set up the runtime environment, ensuring:
- Config is validated before any command logic
- Network/proxy settings are configured before HTTP calls
- Telemetry is deferred (~400KB OpenTelemetry) until actually needed
- OAuth and remote settings load asynchronously without blocking

### 3. MCP Server is a First-Class Mode

`mcp.ts` runs as a **standalone process** via STDIO transport, not within the main CLI:
- Separate process isolation for MCP clients
- Independent LRU file cache (100 entries)
- Zod → JSON Schema conversion for MCP compatibility
- Non-interactive execution context for all tools

### 4. SDK is a Typed Schema-First Foundation

The `sdk/` directory uses **Zod schemas as the source of truth**:
- `coreSchemas.ts` defines all serializable types
- `scripts/generate-sdk-types.ts` auto-generates `coreTypes.generated.js`
- `controlSchemas.ts` defines the bidirectional IPC protocol
- SDK implementations (Python, Rust) use these schemas for type-safe communication

### 5. Stub Implementations in Public SDK

Most functions in `agentSdkTypes.ts` throw `"not implemented"` — the actual logic lives in the host CLI. The SDK provides:
- **Type-safe interfaces** for IDE support
- **Stub functions** that document the expected API
- **Future-proofing** for SDK consumers

### 6. Sandbox Configuration is SDK-Facing Only

`sandboxTypes.ts` defines sandbox settings (filesystem, network, violations) that:
- Are validated at SDK initialization
- Merge with policy settings at runtime
- Support platform-specific features (macOS: Unix sockets, weaker network isolation)

### 7. Profile Checkpoints Throughout

Both `init.ts` and `cli.tsx` call `profileCheckpoint()` at major decision points, enabling granular startup performance analysis.

### 8. Non-Interactive Fallbacks

`mcp.ts` and `init.ts` both have non-interactive modes that:
- Write errors to stderr instead of showing React dialogs
- Call `gracefulShutdownSync()` instead of waiting for async cleanup
- Prevent JSON consumers (marketplace plugins, SDKs) from breaking
