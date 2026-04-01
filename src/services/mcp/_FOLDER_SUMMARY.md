# Summary of `services/mcp/`

## Purpose of `services/mcp/`

This directory implements the **Model Context Protocol (MCP)** client integration for Claude Code, providing authentication, channel management, server orchestration, and cross-account authorization (XAA) for connecting to MCP servers.

---

## Contents Overview

| File | Purpose |
|------|---------|
| `auth.ts` | OAuth 2.0 + PKCE authentication for MCP servers; token storage/refresh/revocation; authorization server discovery |
| `channelAllowlist.ts` | GrowthBook-gated allowlist for which marketplace plugins can register channels |
| `compose.ts` | Message routing and notification forwarding between Claude Code and MCP servers |
| `xaa.ts` | Cross-App Access (XAA) / SEP-990: browser-less OAuth using RFC 8693 Token Exchange + RFC 7523 JWT Bearer Grant |
| `xaaIdpLogin.ts` | OIDC `authorization_code + PKCE` login at the IdP to acquire `id_token`s for XAA |
| `oauthPort.ts` | Port allocation for local OAuth callback servers |
| `types.ts` | TypeScript type definitions for MCP server configs, connections, and channels |
| `utils.ts` | CLI argument parsing, server listing, config writing, MCP URL validation |
| `index.ts` | Entry point: server initialization, lifecycle management, CLI command wiring |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               Claude Code CLI                                │
│                          claude mcp {install|remove|...}                     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               index.ts                                       │
│  • initServer / restartServer / removeServer / listServers                  │
│  • Provides parsed McpServerConfig[] to other modules                        │
└──────────────┬──────────────────────────────────┬──────────────────────────┘
               │                                  │
               ▼                                  ▼
┌──────────────────────────┐         ┌──────────────────────────────────────┐
│      channelAllowlist.ts  │         │         auth.ts                      │
│  • getChannelAllowlist()  │         │  • OAuth 2.0 + PKCE auth flow         │
│  • isChannelAllowlisted() │         │  • Token storage (secureStorage)      │
│  • isChannelsEnabled()     │         │  • Token refresh with file locking   │
│                            │         │  • RFC 8414/9728 discovery            │
└────────────────────────────┘         └──────────────────┬───────────────────┘
                                                          │
                                                          ▼
                                           ┌──────────────────────────────┐
                                           │       xaa.ts                   │
                                           │  • Browser-less OAuth flow     │
                                           │  • Token Exchange (RFC 8693)   │
                                           │  • JWT Bearer Grant (RFC 7523) │
                                           │  • Discovers PRM + AS metadata │
                                           └───────────────┬────────────────┘
                                                           │
                                                           ▼
                                           ┌──────────────────────────────┐
                                           │     xaaIdpLogin.ts            │
                                           │  • OIDC authorization_code     │
                                           │  • Acquires id_token from IdP  │
                                           │  • Caches id_token per issuer  │
                                           └──────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                               compose.ts                                      │
│  • MCPMessageComposer: bidirectional JSON-RPC routing                         │
│  • VSCodeEventNotificationRouter: forwards MCP logs/events to Claude Code    │
│  • AutoModeConfigGate: Statsig-gated autoMode config transmission             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Authentication flows** (`auth.ts` → `xaa.ts` → `xaaIdpLogin.ts`):

| Scenario | Flow |
|----------|------|
| Standard OAuth | `auth.ts` handles full `authorization_code + PKCE` flow directly with AS |
| XAA (browser-less) | `auth.ts` delegates to `xaa.ts`, which chains `xaaIdpLogin.ts` (id_token) → RFC 8693 exchange → RFC 7523 grant |

---

## Key Takeaways

1. **Two authentication paths**: Standard OAuth (`auth.ts`) uses a local HTTP server for browser-based PKCE. XAA (`xaa.ts` + `xaaIdpLogin.ts`) eliminates the browser by exchanging an already-acquired `id_token` through RFC 8693/7523 grants.

2. **Secure token storage**: All tokens (OAuth access tokens, IdP id_tokens) are stored via `secureStorage` with encryption at rest. XAA id_tokens are cached with a 60-second pre-expiry buffer.

3. **File locking on refresh**: Concurrent token refresh attempts are serialized using `lockfile.lock()` with exponential backoff, preventing race conditions when multiple Claude Code processes share an MCP server.

4. **OIDC discovery is cached**: Authorization server metadata (RFC 8414) and protected resource metadata (RFC 9728) are cached in-memory per session and persisted to secure storage for cross-session reuse.

5. **Error normalization**: Non-standard OAuth error responses (e.g., Slack returning HTTP 200 with JSON `error`) are normalized to RFC 6749 format. XAA distinguishes between retryable 5xx and fatal 4xx to decide whether to clear cached tokens.

6. **Channel registration is allowlist-gated**: Plugins must be on the `tengu_harbor_ledger` GrowthBook allowlist and `tengu_harbor` must be enabled for `--channels` to function. This decouples channel rollout from the main MCP feature.

7. **VSCode IPC integration**: `compose.ts` routes MCP lifecycle events back to Claude Code via the `compose/notify` protocol, enabling auto mode config synchronization and VSCode telemetry forwarding through the MCP bridge.

8. **30-second request timeout**: All OAuth/HTTP operations use a 30-second timeout, composed with user cancellation via `AbortSignal.any()` so that Esc key (user cancel) is not clobbered by a lingering timeout.
