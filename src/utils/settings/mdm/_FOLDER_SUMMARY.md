# Summary of `utils/settings/mdm/`

## Purpose of `mdm/`

Provides Mobile Device Management (MDM) integration for Claude Code, enabling enterprise IT departments to push mandatory settings to managed devices. The directory implements a cross-platform abstraction layer that reads MDM configuration from:

- **macOS**: Plist files in `/Library/Managed Preferences/` via `plutil`
- **Windows**: Registry keys under `HKLM/HKCU\SOFTWARE\Policies\ClaudeCode`
- **Linux**: JSON files at `/etc/claude-code/managed-settings.json` and `managed-settings.d/*.json`

## Contents Overview

| File | Purpose |
|------|---------|
| `constants.ts` | Shared configuration constants (paths, timeouts, registry keys) — zero heavy imports for safe cross-module usage |
| `rawRead.ts` | Subprocess orchestration — fires `plutil` or `reg query` commands in parallel with configurable timeouts |
| `settings.ts` | Policy enforcement — parses raw output, validates against schema, applies first-source-wins priority, exposes cache API |

## How Files Relate to Each Other

```
main.tsx (entry)
    │
    ├─ startMdmRawRead() ──────────────┐
    │   └─ fireRawRead() ───────────────┼─ rawRead.ts
    │       └─ Promise.all(             │   Uses:
    │           plutil/reg query)       │   • PLUTIL_PATH
    │                                    │   • PLUTIL_ARGS_PREFIX
    ├─ startMdmSettingsLoad() ──────────┼─ • MDM_SUBPROCESS_TIMEOUT_MS
    │   └─ getMdmRawReadPromise() ───────┤   • Registry paths
    │       └─ consumeRawReadResult() ──┼─ constants.ts
    │           └─ parseCommandOutputAsSettings()
    │               └─ (validates against schema)
    │
    └─ getMdmSettings() ────────────────┘
        └─ mdmCache / hkcuCache ────────── settings.ts
```

**Dependency chain (forward):**
- `settings.ts` → `rawRead.ts` → `constants.ts`
- `settings.ts` → `constants.ts` (direct import for registry paths)

**Initialization sequence:**

1. `constants.ts` loads first — pure config, no side effects
2. `rawRead.ts` exposes `fireRawRead()` and `startMdmRawRead()` — subprocess spawning
3. `settings.ts` calls `startMdmRawRead()` during module evaluation to pre-fetch MDM settings
4. `settings.ts` exposes `ensureMdmSettingsLoaded()` for consumers that need synchronous access

## Key Takeaways

- **Zero heavy imports in `constants.ts`**: Designed for safe import from any module, including `rawRead.ts`, preventing circular dependency issues
- **Parallel subprocess execution**: Both macOS plist reads and Windows registry queries use `Promise.all()` for concurrent execution
- **First-source-wins policy**: Combines multiple MDM sources (plists, HKLM, managed-settings.json, HKCU) with clear priority hierarchy
- **Event loop safety**: `startMdmRawRead()` fires subprocess spawns at module evaluation time before the event loop polls, ensuring non-blocking startup
- **Fault-tolerant validation**: Individual invalid permission rules are filtered rather than rejecting entire MDM configuration
- **Platform-aware defaults**: Windows HKCU provides user-writable lowest-priority settings; macOS/Linux have equivalent fallback paths
