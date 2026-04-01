# Summary of `server/`

## Purpose of `server/`

The `server/` directory implements a WebSocket-based communication layer that enables the SDK to connect to a running Claude Code server. It handles session creation, message routing, permission management, and connection lifecycle—providing both a direct-connect mode (establishes its own session) and a manager for ongoing WebSocket communication with the server.

## Contents Overview

| File | Purpose | Key Exports |
|------|---------|-------------|
| `types.ts` | Defines TypeScript types and Zod schemas | `ServerConfig`, `SessionInfo`, `SessionState`, `connectResponseSchema`, `SessionIndex` |
| `createDirectConnectSession.ts` | Creates a session via HTTP POST | `DirectConnectError`, `createDirectConnectSession()` |
| `directConnectManager.ts` | Manages WebSocket connection lifecycle | `DirectConnectConfig`, `DirectConnectSessionManager`, `DirectConnectCallbacks` |

## How Files Relate to Each Other

```
createDirectConnectSession()                 types.ts
        │                                           │
        │ returns DirectConnectConfig               │ defines types
        ▼                                           ▼
directConnectManager.ts ◄──────────────────────────┘
   │
   │ uses connectResponseSchema()
   │ from lazySchema wrapper
   ▼
WebSocket Connection
   │
   ├── Receives: SDK messages, permission requests
   ├── Sends: User messages, permission responses, interrupts
   └── Routes: via DirectConnectCallbacks interface
```

**Initialization flow:**

1. `types.ts` defines `connectResponseSchema` (lazy-loaded) and all shared types used throughout the module
2. `createDirectConnectSession()` POSTs to `${serverUrl}/sessions` with cwd and permission settings, validates the response with `connectResponseSchema`, and returns a `DirectConnectConfig`
3. `DirectConnectSessionManager` receives the config, opens a WebSocket using the `wsUrl`, and handles bidirectional message flow via callback interfaces

**Data transformation chain:**

```
HTTP POST Response (snake_case)     WebSocket Messages (SDK format)
session_id  ─────────────────────►  sessionId
ws_url      ─────────────────────►  wsUrl
work_dir    ─────────────────────►  workDir
                                       │
                                       ▼
                              SDKUserMessage (outbound)
                              SDKControlResponse (permission)
                              SDKControlRequest (interrupt)
```

## Key Takeaways

- **Two connection modes**: The server supports standard sessions (managed via `RemoteSessionManager`) and direct-connect sessions (handled here), allowing the SDK to bypass CLI invocation
- **Zod validation with lazy loading**: Response schemas use `lazySchema()` to avoid circular import issues at module initialization
- **WebSocket compatibility**: Messages are serialized/deserialized using `jsonStringify`/`jsonParse` to match the exact format expected by `stream-json` input processing on the server side
- **Permission delegation**: Permission requests (`can_use_tool`) are forwarded to the SDK via callbacks, which responds with allow/deny; unsupported subtypes receive error responses to prevent deadlocks
- **Session resumption**: `SessionIndex` persists to `~/.claude/server-sessions.json`, enabling sessions to survive server restarts
- **Graceful shutdown**: `DirectConnectSessionManager` sends interrupt requests via `crypto.randomUUID()` request IDs when closing, ensuring the server can clean up pending operations
