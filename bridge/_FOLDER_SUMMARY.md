# Summary of `bridge/`

## Purpose of `bridge/`

The `bridge/` directory implements the infrastructure for **`claude remote-control`** — a feature that connects local Claude Code development environments to claude.ai's remote control sessions. It handles environment registration, work polling, session spawning, OAuth authentication, trusted device tokens, heartbeat maintenance, and permission event delivery. In essence, it bridges the local CLI process with the cloud backend's environments API, enabling users to control their local environment from the web UI.

## Contents Overview

The directory contains **8 core files** plus supporting subdirectories:

| File | Role |
|------|------|
| **`bridgeApi.ts`** | Low-level HTTP client wrapping the environments REST API. All API methods (register, poll, ack, stop, deregister, archive, reconnect, heartbeat, permission events) live here. Includes OAuth retry logic and error classification. |
| **`bridgeConfig.ts`** | Configuration aggregation layer. Provides `getBridgeAccessToken()` and `getBridgeBaseUrl()` by falling back to OAuth store/config, with **ant-only** env-var overrides for local development. |
| **`bridgeDebug.ts`** | Fault injection framework for testing recovery paths. Allows queuing simulated fatal or transient errors on specific API calls to verify resilience — active only in `ant`-mode builds. |
| **`workSecret.ts`** | Decodes and validates base64url-encoded work secrets from the backend. Builds WebSocket SDK URLs (with protocol routing for localhost vs. production), handles tagged session ID comparisons, and registers as a worker for CCR v2 sessions. |
| **`types.ts`** | All shared TypeScript types, interfaces, and constants. Defines `BridgeConfig`, `BridgeApiClient`, `SessionSpawner`, `SessionHandle`, `WorkSecret`, `WorkResponse`, and UI-related interfaces like `BridgeLogger`. |
| **`index.ts`** | Entry point that exports the `createBridgeApiClient` factory and `isExpiredErrorType` utility. |
| **`debugUtils.ts`** / **`debugUtils.js`** | Debug logging helpers (`logForDebugging`, `debugBody`, `extractErrorDetail`) and stub for `makeApiCall`. |
| **`workSecret.ts`** | Secret decoding and URL construction utilities (see above). |

## How Files Relate to Each Other

```
CLI entry point (/remote-control command)
         │
         ▼
┌─────────────────────────┐
│   bridgeConfig.ts        │  Reads: OAuth store, env vars (ant-only overrides)
│   (getBridgeAccessToken) │
│   (getBridgeBaseUrl)    │
└───────────┬─────────────┘
            │ tokens + base URL
            ▼
┌───────────────────────────────────────┐
│   bridgeApi.ts                        │  Creates BridgeApiClient
│   (createBridgeApiClient)             │  Uses: OAuth retry, error handling
└───────────┬───────────────────────────┘
            │ wrapped API client
            ▼
┌───────────────────────────────────────┐
│   workSecret.ts                       │  Decodes / validates secrets
│   (decodeWorkSecret)                  │  Builds WebSocket + HTTP URLs
│   (registerWorker for CCR v2)        │  Handles tagged session IDs
└───────────┬───────────────────────────┘
            │ decoded secrets, session URLs
            ▼
┌───────────────────────────────────────┐
│   types.ts                            │  Shared types + interfaces
│   (BridgeConfig, SessionSpawner, etc) │  Not a runtime module
└───────────┬───────────────────────────┘
            │
            ▼
    Runtime bridge code (spawner, daemon, REPL)
         │
         ▼
┌───────────────────────────────────────┐
│   bridgeDebug.ts                       │  Fault injection wrapper
│   (wrapApiForFaultInjection)           │  Injects errors for testing
└───────────────────────────────────────┘
```

**Key dependency chain:**
1. `bridgeConfig` is the **configuration source** — all other modules read from it
2. `bridgeApi` is the **HTTP transport layer** — built on top of `bridgeConfig`
3. `workSecret` **transforms backend responses** into typed secrets and URLs
4. `types` **documents the contracts** without importing runtime code
5. `bridgeDebug` **wraps the transport** for fault injection testing (ant-only)

## Key Takeaways

| Category | Detail |
|----------|--------|
| **Architecture** | Layered design: config → transport → transform → runtime. Each layer has a single responsibility and no circular dependencies. |
| **Auth model** | Two-tier: OAuth tokens (primary) + trusted device tokens (CCR v2 elevation). Token sources: keychain (OAuth), macOS keychain (trusted device), env vars (ant-only dev). |
| **OAuth retry** | Single 401 → token refresh → retry. If caller omits `onAuth401` (env-var token users), 401 throws immediately. Prevents circular deps by injecting the refresh callback. |
| **CCR v2 support** | `use_code_sessions: true` in `WorkSecret` triggers CCR v2 code sessions. `workSecret.ts` builds HTTP URLs, `trustedDevice.ts` sends elevated tokens, `registerWorker()` creates the worker epoch for heartbeat auth. |
| **Fault injection** | `bridgeDebug.ts` intercepts poll, register, reconnect, heartbeat calls to inject fatal (`BridgeFatalError`) or transient (axios-style rejection) errors — verified recovery paths without real failures. |
| **ID validation** | `SAFE_ID_PATTERN` (`^[a-zA-Z0-9_-]+$`) on all server-provided IDs used in URL paths — prevents path traversal injection. |
| **Session lifecycle** | `reuseEnvironmentId` in config enables `--session-id` resume by reconnecting to existing backend environment. `consecutiveEmptyPolls` tracking reduces log noise on idle. |
| **Platform-specific** | Trusted device enrollment reads hostname (`process.platform`) for `display_name`. Secure storage spawns macOS `security` subprocess — macOS only. |
| **Ant-only guard** | `bridgeConfig` env var overrides require `USER_TYPE === 'ant'`. `bridgeDebug` is always present but only active in ant builds. |
