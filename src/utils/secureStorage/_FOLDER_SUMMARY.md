# Summary of `utils/secureStorage/`

## Purpose of `utils/secureStorage/`

This directory provides a platform-adaptive, abstraction-layered system for persisting OAuth tokens and API keys securely across Claude Code sessions. It abstracts the underlying storage mechanism (macOS Keychain, plaintext files) behind a unified `SecureStorage` interface, with support for cross-platform fallback, startup prefetching, and graceful degradation.

## Contents Overview

```
utils/secureStorage/
├── types.ts                      # Interface & type definitions
├── plainTextStorage.ts           # Plaintext JSON file backend
├── macOsKeychainHelpers.ts       # Shared caching & helpers (must be lightweight)
├── macOsKeychainStorage.ts       # macOS Keychain backend (via security CLI)
├── keychainPrefetch.ts           # Parallel prefetch for startup optimization
├── fallbackStorage.ts            # Primary-with-fallback wrapper
└── index.ts                      # Platform-aware factory (entry point)
```

| File | Role | Lines |
|------|------|-------|
| `types.ts` | Defines `SecureStorage` interface & `SecureStorageData` type | ~35 |
| `plainTextStorage.ts` | Stores credentials as `.credentials.json` with `0o600` permissions | ~65 |
| `macOsKeychainHelpers.ts` | Lightweight cache + service-name generation (no heavy deps) | ~130 |
| `macOsKeychainStorage.ts` | Reads/writes macOS Keychain via `security` CLI + hex encoding | ~270 |
| `keychainPrefetch.ts` | Parallelizes keychain reads at startup to eliminate blocking | ~130 |
| `fallbackStorage.ts` | Tries primary, falls back to secondary; handles migration/stale cleanup | ~110 |
| `index.ts` | Returns appropriate storage based on `process.platform` | ~15 |

## How Files Relate to Each Other

```
index.ts (platform factory)
  ├── darwin ──► fallbackStorage.ts
  │                 ├── primary: macOsKeychainStorage.ts
  │                 │                 └── macOsKeychainHelpers.ts
  │                 └── secondary: plainTextStorage.ts
  │
  └── linux/other ──► plainTextStorage.ts

keychainPrefetch.ts ──► macOsKeychainHelpers.ts (cache priming only)
                        └── types.ts (type imports)

macOsKeychainStorage.ts ──► macOsKeychainHelpers.ts
                              └── types.ts

fallbackStorage.ts ──► types.ts
plainTextStorage.ts ──► types.ts
```

**Dependency constraints**: `macOsKeychainHelpers.ts` is deliberately lightweight — it must not import `execa`, `human-signals`, or `cross-spawn` because `keychainPrefetch.ts` evaluates it before the main thread unblocks during startup.

## Key Takeaways

1. **Interface-driven design**: All backends conform to `SecureStorage` (`read`, `readAsync`, `update`, `delete`), enabling transparent swaps between plaintext, Keychain, and fallback modes.

2. **macOS dual-backend**: Uses Keychain as primary (secure) with plaintext as fallback (reliable), managed by `fallbackStorage.ts`. Write migration cleans up plaintext after first Keychain write; read migration deletes stale Keychain entries that shadow fresh plaintext data.

3. **Startup prefetch optimization**: `keychainPrefetch.ts` fires Keychain reads in parallel with module evaluation via `startupProfiler.ts:5`, reducing OAuth-ready latency from ~65ms (sequential) to near-zero.

4. **Stale-while-error caching**: `macOsKeychainStorage.ts` serves stale cache on read errors to prevent "Not logged in" states from cascading across all subsystems when the Keychain is temporarily unavailable.

5. **Security hygiene**: Plaintext backend enforces `0o600` file permissions immediately after write; Keychain backend prefers `security -i` (stdin mode) to avoid credential payloads appearing in process-monitor logs.
