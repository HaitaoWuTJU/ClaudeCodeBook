# Summary of `services/remoteManagedSettings/`

## Purpose of `remoteManagedSettings/`

This directory implements a complete **remote managed settings subsystem** for Claude Code, enabling enterprise administrators to push configuration to users' machines via a remote API. The subsystem fetches, caches, validates, and applies server-managed settings while gracefully degrading on errors and respecting security boundaries.

## Contents Overview

| File | Role |
|------|------|
| `types.ts` | Zod schemas and TypeScript types for API responses and fetch results |
| `syncCacheState.ts` | Leaf state module — session cache, eligibility flag, disk read/write |
| `syncCache.ts` | Eligibility checking — determines if user qualifies for remote settings |
| `securityCheck.tsx` | Security validation and user dialog — blocks dangerous new settings |
| `index.ts` | Core service — fetch orchestration, retry logic, caching, background polling |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────┐
│  bootstrap/init.ts                                                      │
│  (entry point)                                                          │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  syncCache.ts  ← syncCacheState.ts (leaf cache management)              │
│  isRemoteManagedSettingsEligible()                                      │
│  • Returns boolean (cached in module-level variable)                   │
│  • Exported for use by bootstrap and other modules                      │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
                ▼               │               │
┌───────────────────────────┐   │               │
│  syncCacheState.ts        │   │               │
│  • getRemoteManaged...     │   │               │
│    FromCache()             │   │               │
│  • setSessionCache()       │   │               │
│  • loadSettings() → disk   │   │               │
│  • getSettingsPath()       │   │               │
└───────────────────────────┘   │               │
                                ▼               │
                    ┌───────────────────────┐   │
                    │  types.ts              │   │
                    │  RemoteManaged...      │   │
                    │    SettingsResponse    │   │
                    │  RemoteManaged...      │   │
                    │    SettingsFetchResult │   │
                    └───────────────────────┘   │
                                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│  index.ts                                                              │
│  loadRemoteManagedSettings()                                           │
│    │                                                                   │
│    ├─► fetchRemoteManagedSettings()  ← axios POST /api/.../settings    │
│    │     ├─► computeChecksumFromSettings()  (SHA256, Python-compatible)│
│    │     └─► fetchWithRetry()  (6 attempts, exponential backoff)       │
│    │                                                                   │
│    ├─► securityCheck.tsx                                               │
│    │     checkManagedSettingsSecurity()                                │
│    │     └─► ManagedSettingsSecurityDialog (Ink/CLI UI)                │
│    │           • Blocks if new settings add dangerous configs          │
│    │           • Logs analytics events                                 │
│    │           • gracefulShutdownSync() on rejection                   │
│    │                                                                   │
│    ├─► syncCacheState.ts                                               │
│    │     setSessionCache() → saveSettings() → notifyChange()           │
│    │                                                                   │
│    └─► startBackgroundPolling()                                        │
│          pollRemoteSettings() every 60 min                             │
└───────────────────────────────────────────────────────────────────────┘
```

**Dependency cycle broken by:**
- `syncCacheState.ts` is a **leaf** — imports only other leaves (`envUtils`, `fileRead`, `jsonRead`, `settingsCache`, `slowOperations`)
- `types.ts` uses `lazySchema()` to defer `SettingsSchema` evaluation, avoiding circular type imports

## Key Takeaways

### Architecture
- **Fail-open design**: If remote settings cannot be fetched and no stale cache exists, the CLI starts normally without server-managed configuration
- **Ephemeral-by-default, persistent-on-demand**: Settings are cached in-memory (`sessionCache`) and optionally persisted to disk only when fetched fresh from the API; disk reads happen only when restarting mid-session
- **Background polling**: Hourly `setInterval().unref()` refreshes settings without blocking CLI usage or process exit

### Security Model
- New settings with dangerous configurations trigger a **blocking dialog** requiring user consent
- Settings are stored with `0o600` permissions (owner read/write only)
- Auth errors skip retry to avoid spamming the API with invalid credentials

### Caching Strategy
- **ETag-style**: Checksum (`"sha256:..."`) sent as `If-None-Match`; `304` responses avoid data transfer
- **Checksum compatibility**: SHA256 with `sort_keys=True, separators=(",", ":")` — matches Python `json.dumps` format used by the server
- **Cache invalidation**: `clearRemoteManagedSettingsCache()` removes both session and file caches; `resetSyncCache()` clears eligibility state

### Authentication
- Two-tier fallback: OAuth Bearer token (for `claude.ai` users) takes priority over API key (for Console users)
- Both require first-party Anthropic infrastructure — custom base URLs and third-party providers are ineligible
