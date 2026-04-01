# Summary of `utils/nativeInstaller/`

# `utils/nativeInstaller/` Directory Summary

## Purpose of `nativeInstaller/`

Provides a complete **file-based native installer system** for Claude Code that handles downloading binaries from multiple sources, installing them with multi-process safety, creating user-accessible symlinks, detecting the originating package manager, and cleaning up stale artifacts. This enables Claude to install and update itself without requiring Node.js or npm at runtime.

## Contents Overview

| File | Role | Key Responsibility |
|------|------|--------------------|
| **`download.ts`** | Download engine | Fetches Claude binaries from Artifactory (ant users) or GCS (external), with SHA256 verification, stall detection, and retry logic |
| **`index.ts`** | Public API (barrel) | Re-exports only the functions/types used by external modules, enforcing a clean interface boundary |
| **`installer.ts`** | Core logic | Manages version installation, atomic moves, symlink creation/cleanup, and orchestrates all cleanup operations |
| **`packageManagers.ts`** | Detection | Identifies which package manager installed the running Claude instance (Homebrew, winget, pacman, dpkg, RPM, APK, mise, asdf) |
| **`pidLock.ts`** | Concurrency control | Provides PID-based version locking with immediate stale-lock detection, replacing the older mtime-based approach |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              External Callers                             │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ imports from
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         index.ts (barrel)                                 │
│  Exports: checkInstall, installLatest, cleanup*, lockCurrentVersion,    │
│           removeInstalledSymlink, SetupMessage                           │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ re-exports from
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       installer.ts (core logic)                          │
│                                                                          │
│  ┌────────────────┐   ┌────────────────────┐   ┌────────────────────┐ │
│  │  Uses download  │   │ Uses pidLock        │   │ Uses packageManagers│ │
│  │  (download.ts)  │   │ (pidLock.ts)        │   │ (packageManagers.ts│ │
│  └────────────────┘   └────────────────────┘   └────────────────────┘ │
│                                                                          │
│  Orchestrates: version resolution → download → verify → install →      │
│                symlink update → cleanup scheduling                       │
└─────────────────────────────────────────────────────────────────────────┘
```

**Dependency chain:**

1. **`installer.ts`** is the central orchestrator — it imports `downloadVersion`, `getLatestVersion` from `download.ts` for fetching binaries
2. **`installer.ts`** uses `withLock`, `acquireProcessLifetimeLock`, `tryAcquireLock`, `cleanupStaleLocks` from `pidLock.ts` to ensure safe concurrent access
3. **`packageManagers.ts`** is queried by `checkInstall()` to report which package manager owns the current installation
4. **`index.ts`** only re-exports from `installer.ts`, serving as the stable public API boundary

## Key Takeaways

1. **Dual download sources**: Artifactory NPM packages for internal users (ant) with `npm ci` integrity checking; GCS buckets for external users with SHA256 verification from `manifest.json`

2. **PID-based locking is a progressive enhancement**: Falls back to 2-hour mtime staleness when PID-based detection is unavailable; controlled via GrowthBook feature flag (`tengu_pid_based_version_locking`)

3. **Atomic installation**: Uses temp file + rename pattern for both lock files and binary installation to avoid corrupted state on crashes

4. **Retention policy**: Keeps only the **2 most recent versions** (`VERSION_RETENTION_COUNT = 2`) and cleans staging directories older than **1 hour**

5. **Platform-aware cleanup**: Detects Linux musl vs glibc via `isMuslEnvironment()`, splits binary names on Windows (`claude.exe`), and handles npm package cleanup differently per OS

6. **Conservative feature rollout**: Test-fixture versions (`99.99.x`) are gated behind a `feature()` macro; external users cannot accidentally install pre-release builds
