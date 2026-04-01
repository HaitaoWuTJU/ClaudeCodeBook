# Summary of `utils/messages/`

## Purpose of `messages/`

The `messages/` subdirectory is the core of the SDK messaging layer, responsible for translating between the application's internal message format and the external SDK interface. It handles:

1. **Initial session handshake** — building the `system/init` message that conveys session metadata to remote clients
2. **Bidirectional message conversion** — transforming messages between internal and SDK types throughout the session lifecycle
3. **Compatibility shims** — bridging naming and structural differences to maintain backward compatibility with SDK consumers

## Contents Overview

| File | Role |
|------|------|
| `systemInit.ts` | Constructs the initial `system/init` SDK message with session metadata (cwd, tools, model, commands, agents, skills, plugins, fast mode) |
| `mappers.ts` | Provides bidirectional mappers for converting messages, compact metadata, rate limit info, and local command output between internal and SDK types |

### `systemInit.ts` — Session Initialization

**Primary Function:** `buildSystemInitMessage(inputs)`

- Emits the first message on the SDK stream for every query turn and on REPL bridge connect
- Collects session state (session ID, API key, cwd, fast mode), tool definitions, MCP clients, commands, agents, skills, plugins, and model settings
- Generates a fresh `uuid` for each init message
- Remaps `'Agent'` tool name to legacy `'Task'` for SDK backward compatibility
- Conditionally adds internal `messaging_socket_path` field when `UDS_INBOX` feature flag is enabled
- Filters commands and skills to only user-invocable entries (`userInvocable !== false`)

### `mappers.ts` — Message Translation Layer

**Primary Functions:**

| Function | Direction | Purpose |
|----------|-----------|---------|
| `toInternalMessages()` | SDK → Internal | Converts SDK messages to internal format; handles assistant/user/compact_boundary system messages |
| `toSDKMessages()` | Internal → SDK | Converts internal messages to SDK format; strips internal-only system messages and command metadata |
| `toSDKCompactMetadata()` / `fromSDKCompactMetadata()` | Internal ↔ SDK | Bidirectional conversion of compact metadata for session resume |
| `toSDKRateLimitInfo()` | Internal → SDK | Maps Claude AI rate limits to SDK format, filtering internal-only fields |
| `localCommandOutputToSDKAssistantMessage()` | Raw → SDK | Converts local command output (e.g., `/voice`, `/cost`) to SDK assistant messages with ANSI stripping and XML tag unwrapping |
| `normalizeAssistantMessageForSDK()` | Internal → SDK | Injects plan content into ExitPlanModeV2 tool inputs for SDK consumers expecting `tool_input.plan` |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                        SDK Consumer                             │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SDKMessage stream                            │
│   system/init ───► assistant/user/system ───► ...                │
└─────────────────────────────────────────────────────────────────┘
           │                      │
           ▼                      ▼
┌──────────────────┐   ┌─────────────────────────────────────────┐
│  systemInit.ts   │   │              mappers.ts                 │
│                  │   │                                         │
│ • Builds init    │◄──│──► toInternalMessages()                 │
│   metadata       │   │     toSDKMessages()                     │
│                  │   │     toSDKCompactMetadata()               │
│                  │   │     fromSDKCompactMetadata()            │
│                  │   │     toSDKRateLimitInfo()                │
└──────────────────┘   └──────────────────────┬──────────────────┘
                                               │
                                               ▼
                               ┌───────────────────────────────┐
                               │     Internal Message Types    │
                               │   Message, AssistantMessage,   │
                               │   UserMessage, SystemMessage,  │
                               │   CompactMetadata             │
                               └───────────────────────────────┘
```

**Relationship:**

1. **`systemInit.ts`** produces the first SDK message (`system/init`) that establishes session context
2. **`mappers.ts`** handles all subsequent message translation throughout the session
3. Both files use shared types from `src/entrypoints/agentSdkTypes.js` for the SDK format
4. Both files reference session state from `src/bootstrap/state.js` for session identifiers
5. The mappers depend on types defined in `src/types/message.js` for internal format

## Key Takeaways

1. **SDK Compatibility is a First-Class Concern**: The `sdkCompatToolName()` function and selective field filtering in mappers demonstrate ongoing effort to maintain backward compatibility while evolving internal types.

2. **Android/Mobile Limitations Drive Workarounds**: Local command output uses `assistant` type instead of a dedicated subtype because Android's SDK message handler doesn't support it, and API-Go's session ingress only handles specific event types.

3. **Metadata Leakage Prevention**: System messages containing command input metadata (`<command-name>`) are deliberately excluded from SDK output to avoid exposing internal details to RC web UI.

4. **Plan Injection for SDK Consumers**: ExitPlanModeV2 tool inputs are augmented with plan content because V2 reads the plan from disk internally, but SDK users expect `tool_input.plan` to be present.

5. **Feature-Gated Functionality**: The UDS messaging socket path is conditionally included based on the `UDS_INBOX` feature flag, allowing internal-only capabilities without affecting public SDK behavior.

6. **ANSI/XML Sanitization**: Command output strings are cleaned of ANSI color codes and XML wrapper tags before being sent to SDK consumers, ensuring consistent cross-platform formatting.
