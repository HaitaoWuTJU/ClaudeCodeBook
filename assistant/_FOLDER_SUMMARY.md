# Summary of `assistant/`

## Purpose of `assistant/`

The `assistant/` directory contains the core **Teleport Agent SDK** — a TypeScript/JavaScript library that enables programmatic interaction with Teleport's API for session management and event retrieval. It supports both Node.js and browser environments and provides paginated access to session history.

## Contents Overview

| File/Directory | Purpose |
|----------------|---------|
| `constants/oauth.js` | OAuth configuration constants (base URL, token endpoint, scopes) |
| `entrypoints/agentSdk.js` | Main SDK entrypoint (`createSdk`) — orchestrates auth and API calls |
| `entrypoints/agentSdkTypes.js` | TypeScript type definitions for all SDK interfaces |
| `utils/debug.js` | Debug logging utility (respects `DEBUG` environment variable) |
| `utils/session/history.js` | Session history retrieval with pagination support |
| `utils/teleport/api.js` | Core API utilities — OAuth headers, request preparation, token refresh |

## How Files Relate to Each Other

```
agentSdk.js (createSdk)
    ├── oauth.js (getOauthConfig)
    ├── api.js
    │       ├── getOAuthHeaders() → uses oauth.js config
    │       ├── prepareApiRequest() → validates + refreshes OAuth token
    │       └── apiRequest() → makes HTTP requests
    └── history.js
            └── createHistoryAuthCtx() → uses api.js helpers
                    └── fetchPage() → uses apiRequest()
```

**Dependency chain:**

1. `createSdk()` in `agentSdk.js` calls `getOauthConfig()` to get the base URL
2. It then uses `prepareApiRequest()` to validate tokens and attach OAuth headers
3. `history.js` wraps these utilities to provide `fetchLatestEvents()` and `fetchOlderEvents()`
4. `debug.js` is used by both `api.js` and `history.js` for logging on failures

## Key Takeaways

- **OAuth 2.0 flow**: The SDK handles token acquisition, storage, and automatic refresh via PKCE
- **Dual environment support**: Detects Node.js vs browser and uses appropriate HTTP clients (`node-fetch` vs `fetch`/`axios`)
- **Type safety**: Comprehensive TypeScript types exported via `agentSdkTypes.js`
- **Error resilience**: Requests use 15-second timeouts and graceful error handling that returns `null` rather than throwing
- **Pagination**: Session history supports cursor-based pagination with `anchor_to_latest` and `before_id` parameters
- **Debugging**: Optional debug logging via `DEBUG` environment variable with namespaced output
- **Custom headers**: Requests include `anthropic-beta: ccr-byok-2025-07-29` and `x-organization-uuid` for beta feature access
