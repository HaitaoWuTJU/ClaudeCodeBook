# Summary of `services/api/`

## `services/api/` Directory Summary

### Purpose

This directory provides a unified **API client layer** for Claude Code, encapsulating all HTTP communication with:
- **Anthropic's Claude API** (core LLM interactions)
- **Claude AI OAuth endpoints** (subscription, usage, quotas, bootstrap data)
- **Teleport API endpoints** (organization management, billing, user permissions, admin requests)

It abstracts authentication, error handling, retry logic, caching, and response transformation behind typed utility functions used throughout the application.

---

### Contents Overview

| File | Responsibility |
|------|----------------|
| **`claude.ts`** | Primary Anthropic API client: streaming/non-streaming requests, tool execution, prompt caching, model fallback, token budget management |
| **`withRetry.ts`** | Exponential backoff retry wrapper (`withRetry<T>()`) with support for 401 refresh, 529/429 rate limits, context overflow, cloud credential rotation, fast-mode cooldown, and persistent/unattended mode with heartbeat yields |
| **`http.ts`** | HTTP utility layer: `getAuthHeaders()` (OAuth or API key), `withOAuth401Retry()` (401 → token refresh → retry) |
| **`index.ts`** | Barrel re-export of all public API utilities for convenient imports |
| **`bootstrap.ts`** | Fetches Claude CLI bootstrap config (`/api/claude_cli/bootstrap`), validates with Zod, persists to disk cache only on changes |
| **`usage.ts`** | Fetches real-time utilization data from `/api/oauth/usage` (5h/7d/extra usage windows) for subscriber dashboards |
| **`ultrareview.ts`** | Fetches UltraReview quota (`reviews_used`, `reviews_limit`, `is_overage`) from `/v1/ultrareview/quota` |
| **`teleport/api.ts`** | Teleport OAuth helpers: `prepareApiRequest()`, `getOAuthHeaders()`, organization/user resolution |
| **`teleport/billing.ts`** | Teleport billing subscriptions, plan details, and renewal eligibility |
| **`teleport/organizations.ts`** | Teleport organization listing, UUID resolution, and current org info |
| **`teleport/users.ts`** | Teleport user roles (`isBillingAdmin`, `isOrganizationAdmin`), permissions checks, and user details |
| **`teleport/adminRequests.ts`** | Teleport admin requests (limit increases, seat upgrades) with eligibility checking |
| **`rateLimitMocking.js`** | Internal testing hook that intercepts rate limit responses |

---

### How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEPENDENCY HIERARCHY                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LAYER 3 — High-Level Domain APIs (user-facing business logic)              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│  │  bootstrap.ts    │  │  usage.ts        │  │  ultrareview.ts  │         │
│  │  (config cache)  │  │  (utilization)   │  │  (quota)         │         │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘         │
│           └──────────────────────┼──────────────────────┘                  │
│                                  ▼                                          │
│  LAYER 2 — Teleport Domain APIs                                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │  billing   │  │  orgs      │  │  users     │  │  adminReqs │           │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘           │
│        └───────────────┼───────────────┼───────────────┘                  │
│                        ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐               │
│  │                  teleport/api.ts                        │               │
│  │   prepareApiRequest(), getOAuthHeaders()                │               │
│  └─────────────────────────┬───────────────────────────────┘               │
│                            │                                                │
│  LAYER 1 — Core HTTP & Retry Infrastructure                                 │
│  ┌─────────────────────────┼───────────────────────────────┐                │
│  │              http.ts    │    withRetry.ts              │                │
│  │  getAuthHeaders()       │  withRetry<T>()              │                │
│  │  withOAuth401Retry()    │  Exponential backoff         │                │
│  └───────────┬─────────────┴───────────────┬───────────────┘                │
│              │                             │                                │
│              ▼                             ▼                                │
│  ┌─────────────────────────────────────────────────────────┐                │
│  │                    claude.ts                            │                │
│  │  queryHaiku(), queryWithModel(), getClient()            │                │
│  │  (streams, tools, cache breaks, model fallback)         │                │
│  └─────────────────────────┬───────────────────────────────┘                │
│                            │                                                │
│  LAYER 0 — I/O Primitives (imported, not owned by this directory)           │
│  ├── axios          (HTTP client)                                          │
│  ├── src/utils/auth.js     (OAuth tokens, subscriber check, scopes)        │
│  ├── src/utils/sleep.js     (abort-aware sleep)                              │
│  └── src/utils/config.js    (global config read/write)                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Cross-cutting concerns:**
- **`http.ts`** is the shared foundation — both `teleport/api.ts` (indirectly via `prepareApiRequest`) and `claude.ts` use its `getAuthHeaders()` and `withOAuth401Retry()`
- **`withRetry.ts`** wraps both the Anthropic SDK calls in `claude.ts` and all Teleport domain calls, providing consistent retry behavior across the entire API layer
- **`teleport/api.ts`** acts as the **Teleport-specific HTTP builder**, reading OAuth tokens and constructing headers/URLs used by all Teleport domain files (`billing`, `organizations`, `users`, `adminRequests`)

---

### Key Takeaways

1. **Dual authentication paths**: The API layer supports both OAuth tokens (`getOAuthHeaders`) and API keys (`getAuthHeaders`), with OAuth tokens triggering automatic 401 → refresh → retry on expiry.

2. **Tiered retry logic**: `withRetry<T>()` is the central retry engine — it handles Anthropic rate limits (429/529), context overflow (adjusting `max_tokens`), cloud credential rotation (AWS Bedrock, GCP Vertex), and persistent mode with 30-second heartbeats for long-running unattended sessions.

3. **Cache-first design for startup**: `bootstrap.ts` uses deep equality (`isEqual`) to skip disk writes when cached data hasn't changed, preventing unnecessary config writes on every Claude Code startup.

4. **Teleport as a separate API domain**: Teleport operations use a distinct `prepareApiRequest()` flow (with `x-organization-uuid` headers) separate from the main Claude API client, reflecting that Teleport is a self-hosted/enterprise product with its own API surface.

5. **Graceful degradation throughout**: Every external API call returns `null` or `{}` on failure rather than throwing, enabling callers to handle missing data (e.g., non-subscribers see empty quotas, unauthenticated users get `null` usage data).
