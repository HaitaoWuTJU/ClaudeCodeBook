# Summary of `remote/`

## Purpose of `remote/`

The `remote/` directory implements the **remote execution layer** for Claude Code, enabling the CLI to delegate code execution, tool use, and LLM inference to a remote CCR (Cloud Code Runtime) container. This architecture allows Claude Code to run in environments where local tool execution is unavailable (e.g., restricted shells, sandboxed environments).

The core responsibilities are:

- **Transport abstraction**: WebSocket for receiving messages, HTTP for sending user input
- **Session management**: Connection lifecycle, authentication, reconnection
- **Message translation**: Converting between SDK protocol and internal message types
- **Permission bridging**: Creating synthetic messages and tool stubs for remote permission flows

## Contents Overview

| File | Responsibility |
|------|----------------|
| `SessionsWebSocket.ts` | Low-level WebSocket client handling connection, ping/pong keepalive, automatic reconnection with exponential backoff, and JSON serialization |
| `RemoteSessionManager.ts` | Session-level coordinator managing WebSocket subscription, HTTP message sending, permission request/response flow, and connection state callbacks |
| `sdkMessageAdapter.ts` | Message translator converting SDK-format messages from the remote CCR into internal REPL Message/StreamEvent types for rendering |
| `RemoteExecutionManager.ts` | Tool execution layer creating synthetic AssistantMessages and minimal Tool stubs to bridge local permission UI with remote execution |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Remote CCR Container                           │
│   (runs tools, executes LLM, tracks permissions, sends SDK messages)        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       │ WebSocket (subscribe)
                                       │ HTTP POST (send events)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  SessionsWebSocket.ts                                                          │
│  • WebSocket transport layer (Bun native or ws fallback)                     │
│  • Connection management, ping/pong, reconnection                            │
│  • JSON parse/serialize, message routing                                     │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       │ Emits: SDKMessage, control_request,
                                       │        control_response, control_cancel_request
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  RemoteSessionManager.ts                                                       │
│  • Session orchestration layer                                                │
│  • Permission request tracking (Map<request_id, request>)                    │
│  • Routes permission callbacks → local handlers                             │
│  • Sends user messages via HTTP POST                                         │
│  • Sends permission responses via WebSocket                                  │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       │ SDKMessage (from WebSocket)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  sdkMessageAdapter.ts                                                         │
│  • Translates SDK protocol → internal types                                  │
│  • Filters redundant/noisy messages (success results, auth events)          │
│  • Returns: AssistantMessage | SystemMessage | StreamEvent | ignored        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       │ Internal messages
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  REPL Renderer / Permission UI (external consumer)                            │
│  • Displays messages to user                                                 │
│  • Shows permission prompts via onPermissionRequest callback                │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Separate path for tool execution:**

```
Permission Prompt (user approves)
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  RemoteExecutionManager.ts                                                    │
│  • Creates synthetic AssistantMessage for permission confirmation           │
│  • Creates minimal Tool stubs (call → FallbackPermissionRequest)             │
│  • Used when remote has tools (e.g., MCP) unknown to local CLI               │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Two-channel communication**: The remote system uses WebSocket for server-to-client push (messages, permission requests) and HTTP POST for client-to-server requests (user input, permission responses). This mirrors the bidirectional message flow needed for real-time agent interaction.

2. **Graceful degradation**: Both `sdkMessageAdapter` and `SessionsWebSocket` handle unknown message types gracefully—logging and continuing rather than crashing. This enables backward compatibility when the remote adds new message types.

3. **Permission bridging is critical**: `RemoteExecutionManager` creates synthetic messages and stubs because the local CLI may not know about remote-only tools (like MCP tools). The permission UI must still present these to users for approval, even though execution happens remotely.

4. **Reconnection complexity**: `SessionsWebSocket` implements differentiated retry logic:
   - **4001 (session not found)**: Up to 3 retries with linear backoff, handling transient "container starting" delays
   - **4003 (auth failure)**: No retries, immediate failure
   - **Other errors**: Up to 5 retries at fixed intervals

5. **Viewer-only mode**: `RemoteSessionManager` supports `viewerOnly: true` which disables interrupts, extended reconnect timeouts, and title updates—optimized for the `claude assistant` read-only experience.

6. **Tool result detection workaround**: `sdkMessageAdapter` identifies tool results by their content block shape rather than `parent_tool_use_id`, because agent-side ID normalization can corrupt the field.
