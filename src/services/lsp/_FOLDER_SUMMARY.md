# Summary of `services/lsp/`

## Purpose of `services/lsp/`

Implements Language Server Protocol (LSP) integration, enabling Claude Code to provide **editor-like features**—diagnostics, hover information, go-to-definition—powered by real language servers. The directory manages the complete lifecycle of LSP server processes spawned as child subprocesses, routes requests to the correct server based on file type, and delivers diagnostic feedback back to Claude's query response system.

## Contents Overview

```
services/lsp/
├── types.ts                      # Shared TypeScript types
├── config.ts                     # Loads LSP server configs from plugins
├── LSPClient.ts                  # Low-level JSON-RPC ↔ subprocess bridge
├── LSPServerInstance.ts          # Single language server with request handlers
├── LSPServerManager.ts            # Routes requests to servers by file extension
├── manager.ts                    # Singleton lifecycle management
├── LSPDiagnosticRegistry.ts      # Async diagnostic delivery with deduplication
└── passiveFeedback.ts            # Wires publishDiagnostics → Claude attachment system
```

| File | Role |
|------|------|
| `config.ts` | Discovers configured LSP servers from enabled plugins |
| `LSPClient.ts` | Spawns a subprocess and wraps stdio with `vscode-jsonrpc` |
| `LSPServerInstance.ts` | Encapsulates one language server (handles init, requests, notifications, lifecycle) |
| `LSPServerManager.ts` | Routes LSP operations to the correct server based on file extension |
| `manager.ts` | Owns the singleton `LSPServerManager` instance; handles init/reinit/shutdown |
| `LSPDiagnosticRegistry.ts` | Queues, deduplicates, volume-limits, and delivers diagnostics |
| `passiveFeedback.ts` | Intercepts `textDocument/publishDiagnostics` from all servers |

## How Files Relate to Each Other

### Data Flow: Top-Down (Request Path)

```
config.ts
  └─ getAllLspServers()
        ↑ reads plugin registry
        ↓ returns ScopedLspServerConfig[]

manager.ts
  └─ initializeLspServerManager()
        ↓ calls createLSPServerManager()

LSPServerManager.ts
  └─ createLSPServerManager()
        ↓ calls manager.initialize()

        ↓ calls createLSPServerInstance() per server
              ↓ for each server

LSPServerInstance.ts
  └─ createLSPServerInstance()
        ↓ calls createLSPClient()

LSPClient.ts
  └─ createLSPClient()
        ↓ spawns subprocess (child_process.spawn)
        ↓ wraps stdio with vscode-jsonrpc
        ↓ exposes sendRequest(), sendNotification(), onNotification(), onRequest()
```

### Data Flow: Bottom-Up (Diagnostic Path)

```
LSP server subprocess
  └─ textDocument/publishDiagnostics notification
        ↓ via JSON-RPC

LSPClient.ts
  └─ onNotification() handler

LSPServerInstance.ts
  └─ forwards to passiveFeedback.ts handler

passiveFeedback.ts
  └─ formatDiagnosticsForAttachment() + registerPendingLSPDiagnostic()

LSPDiagnosticRegistry.ts
  └─ queues diagnostics, deduplicates, volume-limits
        ↓ async delivery

Query Response (attachment system)
```

### Singleton Ownership

```
manager.ts (singleton owner)
  └─ owns lspManagerInstance
        ↓ returns via getLspServerManager()

passiveFeedback.ts (subscriber)
  └─ reads via manager.getAllServers()
        ↓ registers handlers on each server
```

## Key Takeaways

1. **LSP servers are exclusively plugin-driven.** `config.ts` only discovers servers from enabled plugins—no user/project settings. This means LSP features depend entirely on plugin configuration.

2. **Two-layer error handling isolation.** `manager.ts` uses `Promise.allSettled` for shutdown so one server's failure doesn't abort others. Within each server, `LSPServerInstance` catches and logs errors before they can propagate upward.

3. **Lazy initialization with handler queuing.** `LSPClient.ts` queues handlers registered before `start()` is called and applies them once the connection is ready. This supports registering handlers before the server process exists.

4. **Diagnostic deduplication operates across two dimensions:**
   - **Within a batch** — `createDiagnosticKey()` creates a content-based hash of message, severity, range, source, and code to collapse duplicates
   - **Across turns** — `deliveredDiagnostics` LRU cache tracks what was already shown so identical diagnostics don't reappear after re-querying

5. **Reinitialization fixes a plugin discovery race.** `manager.ts` exposes `reinitializeLspServerManager()` which resolves issue #15521: if the plugin cache is cleared and repopulated (e.g., after marketplace reconciliation), reinitialization picks up newly available servers instead of using the stale cached empty list.

6. **Graceful degradation at every layer.** If LSP initialization fails, the system logs the error and continues without editor features—no hard failure. The `getInitializationStatus()` discriminated union reflects `not-started`, `pending`, `success`, or `failed` so callers can branch accordingly.

7. **Bare mode skips LSP entirely.** `manager.ts` checks `isBareMode()` at startup and returns early, since scripted/non-interactive usage has no need for diagnostics or hover information.

8. **Protocol tracing is built-in.** `LSPClient.ts` enables `vscode-jsonrpc` verbose tracing with output to `logForDebugging()`, useful for diagnosing LSP protocol-level issues.
