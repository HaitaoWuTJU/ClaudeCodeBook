# Summary of `types/`

## Purpose of `types/`

The `types/` directory serves as the **central type definition layer** for the Claude Code CLI tool. It provides comprehensive TypeScript type safety across all major subsystems — command execution, plugin loading, hook interception, permission management, MCP connectivity, text input handling, and session logging. All type-only modules (`.ts` with no runtime code) ensure zero bundle overhead while enabling IDE autocomplete, compile-time error detection, and cross-module type contracts throughout the codebase.

## Contents Overview

| File | Purpose |
|------|---------|
| `hooks.ts` | Defines three command paradigms (PromptCommand, LocalCommand, LocalJSXCommand) with execution modes, lazy-loading signatures, availability constraints, and completion callbacks |
| `hooks.ts` | Hook event system with Zod validation schemas, type guards, discriminated unions of 17 hook event types, permission request/result types, and aggregated hook result merging |
| `ids.ts` | Branded nominal types (`SessionId`, `AgentId`) preventing cross-contamination of identifiers at compile time, with casting helpers and regex-validated factory (`toAgentId`) |
| `logs.ts` | Session logging/transcript system with `LogOption` interface, message type variants (agent session, attribution, file history, MCP, tool classification), and chronological log sorting |
| `mcp.ts` | Model Context Protocol server types — JSON-RPC transport wrappers, request/notification/response envelopes, protocol negotiation, resource subscription, tool calling, sampling callbacks, progress tracking, and lifecycle management |
| `permissions.ts` | Full permission system — modes (`auto` via feature flag), behaviors, rule sources, update operations, allow/ask/deny decisions with generics, bash classifier with fast/thinking stages, and `ToolPermissionContext` for rule evaluation |
| `plugin.ts` | Plugin system — 24-variant discriminated `PluginError` union, `LoadedPlugin` with five component slots, `PluginRepository` metadata, `PluginLoadResult` aggregation, and `getPluginErrorMessage()` for UI formatting |
| `textInputTypes.ts` | Text input primitives — base/Vim input props, ghost text, windowed rendering via viewport offsets, image/text paste handling, three-level `QueuePriority` (`now`/`next`/`later`), `OrphanedPermission` for bridge safety |
| `generated/google/timestamp.ts` | `google.protobuf.Timestamp` — seconds (Long) + nanos representation with JSON converters, Amino/SDK variants, zero-value defaults |

## How Files Relate to Each Other

```
                    ┌──────────────────────┐
                    │      hooks.ts        │
                    │  (Command + HookEvent)│
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
    ┌─────────────────┐  ┌──────────┐   ┌──────────────┐
    │  textInputTypes │  │ permissions│   │    logs.ts   │
    │  (PromptCommand │  │  (HookResult │   │ (HookProgress│
    │   context)      │  │   aggregator│   │  listener)   │
    └─────────────────┘  └──────────┘   └──────────────┘
              │                │                │
              └────────┬───────┘                │
                       │                        │
                       ▼                        │
              ┌──────────────────┐              │
              │     plugin.ts    │              │
              │ (Command source) │◄─────────────┘
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │    mcp.ts        │
              │ (Tool source &   │
              │  notification)  │
              └──────────────────┘

              ┌──────────────────┐
              │     ids.ts       │
              │ (Shared branded  │
              │  ID types)       │
              └────────┬─────────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
     textInputTypes         hooks.ts
     (agentId in          (AgentId in
      QueuedCommand)       HookCallback)
```

**Dependency chain for a typical execution path:**

1. `plugin.ts` loads commands from plugin manifests
2. Commands resolve into `PromptCommand`/`LocalCommand`/`LocalJSXCommand` (from `hooks.ts`)
3. Commands use `ToolPermissionContext` (from `permissions.ts`) for rule evaluation
4. Text input triggers `QueuedCommand` with priority (from `textInputTypes.ts`)
5. Hook callbacks receive `HookResult` (from `permissions.ts`) aggregated from multiple sources
6. Logs flow through `LogOption` (from `logs.ts`) with `timestamp` from `generated/google/timestamp.ts`
7. `SessionId`/`AgentId` (from `ids.ts`) thread through hooks, text input, and logs as correlation keys

## Key Takeaways

1. **Nominal-like ID safety** — `SessionId` and `AgentId` use TypeScript's branded-type pattern to prevent swapping session and agent identifiers, with `toAgentId()` providing regex validation (`/^a(?:.+-)?[0-9a-f]{16}$/`) for external input.

2. **Discriminated unions for exhaustive matching** — Both `PluginError` (24 variants) and `hookSpecificOutput` (17 variants) use discriminated unions enabling TypeScript's exhaustive switch completion and eliminating stringly-typed error handling.

3. **Three execution modes for commands** — Prompt commands support `inline` (conversation integration) and `fork` (isolated sub-agent with separate context/budget), while Local/JSX commands use lazy loading to defer heavy dependencies until invocation.

4. **Lazy schema pattern** — Both `hooks.ts` files wrap Zod schemas in `lazySchema()` to break circular import chains (schemas reference types that reference schemas), deferring validation until first use.

5. **Generics for flexible passthrough** — `PermissionAllowDecision`, `PermissionDenyDecision`, and `HookResult` accept generic type parameters (defaulting to `{ [key: string]: unknown }`) allowing callers to preserve input shapes through permission evaluation without widening.

6. **Queue priority for cooperative scheduling** — `now`/`next`/`later` priorities enable graceful interruption handling: `now` aborts in-flight work, `next` wakes `SleepTool` mid-turn, `later` waits for full turn completion.

7. **Feature-gated permission mode** — The `auto` permission mode is only available when `feature('TRANSCRIPT_CLASSIFIER')` is enabled, using `satisfies` to validate array membership at compile time.

8. **Hook system separates concerns** — `HookCallback` runs synchronously in the CLI process; `AsyncHookJSONOutput` runs in a subprocess with a configurable timeout. The `HookProgress` event flows into session logs for analytics without blocking hook execution.

9. **MCP as a first-class citizen** — `mcp.ts` handles the full JSON-RPC lifecycle (requests, notifications, streaming responses, protocol negotiation, sampling callbacks) rather than delegating to a third-party library, ensuring type safety end-to-end.

10. **JSON serialization for cross-boundary transport** — `Timestamp` uses `Long` for 64-bit seconds (safe past year 2262) and provides `fromJSON`/`toJSON` converters, making it the canonical temporal wire format across all event schemas regardless of environment.
