# Summary of `cli/`

## `cli/` Directory Summary

### Purpose

The `cli/` directory is the **runtime entry point and orchestration layer** for the Claude Code SDK. It owns everything after the CLI dispatcher (`entry.tsx`) decides which subcommand to run: printing, I/O routing, transport negotiation, SDK control message handling, update flows, and command-level handlers.

---

### Contents Overview

| Path | Role |
|------|------|
| **`print.ts`** | Central REPL loop: session management, tool execution, MCP server lifecycle, permission handling, SDK control message processing |
| **`structuredIO.ts`** | Stdio-based structured I/O: reads/writes NDJSON control messages to SDK consumers (VS Code, CCR bridge), races local hooks against remote permission prompts |
| **`remoteIO.ts`** | Remote I/O for SDK mode: HTTP transport over SSE, CCR v2 protocol (heartbeats, state uploads, event buffering, keep-alive, permission response echo) |
| **`update.ts`** | Update workflow: detects installation type, selects update path (native / npm global / npm local / package manager), handles lock contention |
| **`exit.ts`** | Thin exit helpers: `cliError()` and `cliOk()` — centralized `process.exit(1/0)` with optional stdout/stderr messages |
| **`ndjsonSafeStringify.ts`** | Safe NDJSON serialization: escapes U+2028/U+2029 line terminators so JSON values survive line-splitting receivers |
| **`handlers/`** | Lazy-loaded per-subcommand handlers: `auth`, `agents`, `autoMode`, `mcp`, `plugins`, `util` |
| **`transports/`** | Runtime-polymorphic transport layer: `Transport` interface with three implementations (SSE, WebSocket, Hybrid) plus CCR v2 protocol clients |

---

### How Files Relate to Each Other

```
entry.tsx (CLI dispatcher)
         │
         │ dynamic import (lazy)
         ▼
┌─────────────────────────────────────────────────────┐
│                    print.ts                          │
│  ┌──────────────────────────────────────────────┐   │
│  │  Session / tool / permission orchestration   │   │
│  └──────────────────────────────────────────────┘   │
│                       │                              │
│          ┌────────────┴────────────┐                 │
│          ▼                         ▼                 │
│   structuredIO.ts            remoteIO.ts            │
│   (stdio protocol)            (HTTP/SSE protocol)   │
│          │                         │                 │
│          │                    transports/            │
│          │                    (SSE, WebSocket,        │
│          │                     Hybrid, CCRClient)    │
│          ▼                         │                 │
│  ndjsonSafeStringify.ts            │                 │
│  (escapes U+2028/U+2029)           │                 │
│          │                         │                 │
└──────────┼─────────────────────────┼─────────────────┘
           │                         │
           ▼                         ▼
    ┌────────────────┐        ┌────────────────┐
    │ exit.ts        │        │ exit.ts        │
    │ (process.exit) │        │ (process.exit) │
    └────────────────┘        └────────────────┘
```

**Handlers** (`handlers/`) are orthogonal to the I/O layer: they execute their own business logic (auth, plugin management, MCP server setup, update checks) and call `exit.ts` helpers to terminate the process. They share utilities like `analytics`, `config`, `settings`, and `debug` with the core loop but do not participate in structured I/O.

**Transports** are swapped at runtime via `getTransportForUrl()` (in `remoteIO.ts`). The `print.ts` loop is transport-agnostic — it calls `structuredIO` or `remoteIO` based on CLI flags (`--sdk-url`).

---

### Key Takeaways

#### 1. Three I/O modes, one loop
`print.ts` is the **only** file that knows whether it is talking to a local process (`StructuredIO`) or a remote runtime (`RemoteIO`). Every other module is transport-unaware. This keeps the tool execution, permission, and session logic centralized.

#### 2. NDJSON is the universal wire format
Both `structuredIO.ts` (stdio) and `remoteIO.ts` (HTTP) use NDJSON. `ndjsonSafeStringify.ts` patches a subtle JSON↔ECMA-262 discrepancy: raw U+2028/U+2029 are legal in JSON but act as newlines in JavaScript string splitting. All values are serialized through it before crossing the I/O boundary.

#### 3. Permission racing is the core UX differentiator
`structuredIO.ts` races **local permission hooks** (`executePermissionRequestHooks`) against **SDK permission prompts** (remote user approval via `control_request`). The first to resolve wins; the loser is aborted. This gives local rules (e.g., auto-confirm `Read` on specific paths) latency comparable to remote approval while still supporting interactive human decisions.

#### 4. MCP lifecycle is centralized in `print.ts`
MCP server lifecycle (connect, reconfigure, auth, vscode SDK, vs. stdio transport) is fully managed in `print.ts`. Handlers only add/remove servers via SDK `control_request` messages; `print.ts` reconciles desired vs. actual state and enforces enterprise policy filters (`allowedMcpServers`/`deniedMcpServers`) on both the CLI path and the SDK `setMcpServers()` path.

#### 5. `update.ts` is the only file with Homebrew/winget/apk/package-manager logic
All other configuration and lifecycle logic assumes native or npm installations. The update handler carries the full burden of detecting the installation method, presenting appropriate upgrade instructions, and falling back gracefully when locks or network failures occur.

#### 6. `exit.ts` eliminates ~60 copies of `process.exit` + `console.error`
`cliError()` and `cliOk()` are the **sole exit points** for all handlers and the main loop. Centralizing them enables consistent output formatting, testability (spies on `console.error`/`stdout.write`/`process.exit`), and lint suppression (file-level `eslint-disable no-process-exit`).

#### 7. Transport polymorphism uses environment-gated factory dispatch

```
getTransportForUrl()
├── SSE          ← CCR v2 && POST-for-ingress flags
├── Hybrid       ← POST-for-ingress (WebSocket reads, HTTP writes)
└── WebSocket    ← default (with ping/pong, keep-alive, reconnect)
```

`SerialBatchEventUploader` is the shared write engine for HTTP-based transports, enforcing **serialized writes** to prevent Firestore collision (concurrent POSTs to the same document would overwrite each other).
