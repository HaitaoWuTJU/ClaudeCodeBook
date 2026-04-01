# Summary of `commands/logout/`

## Purpose of `logout/`

The `logout/` directory implements a complete user logout command for a CLI application. It orchestrates credential removal, secure storage wiping, cache clearing, telemetry flushing, and graceful shutdown to fully disconnect a user from their Anthropic account.

## Contents Overview

The directory contains two files working together:

| File | Role |
|------|------|
| `logout.ts` | Command configuration descriptor — defines metadata, feature flags, and lazy-loads implementation |
| `logout.tsx` | Actual implementation — contains `performLogout()`, `clearAuthRelatedCaches()`, and `call()` |

## How Files Relate to Each Other

```
logout.ts (descriptor)
    │
    ├── Defines: name="logout", type="local-jsx"
    ├── Defines: isEnabled() checks DISABLE_LOGOUT_COMMAND env var
    │
    └── load() ──imports()──▶ logout.tsx (implementation)
                                │
                                ├── performLogout({ clearOnboarding })
                                │     ├── flushTelemetry() [lazy]
                                │     ├── removeApiKey()
                                │     ├── wipeSecureStorage()
                                │     ├── clearAuthRelatedCaches()
                                │     └── updateGlobalConfig()
                                │
                                └── call()
                                      └── performLogout() + scheduleShutdown()
```

**Flow**: When a user runs `logout`:
1. `logout.ts` is imported → `isEnabled()` checks env → `load()` triggers lazy import
2. `logout.tsx` executes → `call()` runs → `performLogout()` wipes all auth data
3. Success message renders via Ink → 200ms grace period → `gracefulShutdown()`

## Key Takeaways

- **Telemetry-first**: Flushes telemetry *before* clearing credentials to prevent org data leakage
- **Comprehensive wipe**: Clears 9+ distinct caches (OAuth, user, tools, Grove, policy, beta features) to prevent stale auth state
- **Feature-gated**: Entire command can be disabled via `DISABLE_LOGOUT_COMMAND` environment variable
- **Lazy loading**: Keeps startup bundle lean by dynamically importing telemetry flush (~1.1MB savings)
- **Selective reset**: The `clearOnboarding` flag allows partial config clearing (preserves settings, wipes onboarding state)
