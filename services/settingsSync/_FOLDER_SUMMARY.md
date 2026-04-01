# Summary of `services/settingsSync/`

## `settingsSync/` Directory

### Purpose

This directory implements a **bidirectional settings and memory file synchronization service** for Claude Code. It enables user settings (preferences) and memory files (CLAUDE.md) to be shared between the interactive CLI and CCR (Claude Code Remote) environments via a backend API. The service supports:

- **Upload (interactive CLI)**: Incrementally pushes changed local settings to a remote endpoint in the background
- **Download (CCR)**: Pulls remote settings to the local filesystem before plugin installation

### Contents Overview

| File | Role |
|------|------|
| `settingsSync.ts` | Core sync logic — file I/O, API communication, background upload, cache management, feature flags |
| `settingsSync/types.ts` | Zod schemas for API request/response validation and TypeScript type definitions |

### How Files Relate to Each Other

```
settingsSync.ts
    │
    ├── imports UserSyncData, SettingsSyncFetchResult,
    │   SettingsSyncUploadResult, SYNC_KEYS from:
    ▼
settingsSync/types.ts ──Zod──► UserSyncDataSchema
                                       │
                     ┌─────────────────┼─────────────────┐
                     │                 │                 │
                     ▼                 ▼                 ▼
                userId (str)      lastModified       content
                                  (ISO 8601)       (key-value map)
                                  checksums
```

- **`types.ts`** provides the **type contracts** and **Zod schemas** that **`settingsSync.ts`** uses to:
  - Validate incoming API responses with `UserSyncDataSchema.parse(result.data)`
  - Shape return types of `SettingsSyncFetchResult` and `SettingsSyncUploadResult`
  - Reference canonical sync key constants (`SYNC_KEYS.userSettings`, `SYNC_KEYS.userMemory`, etc.)

- **`settingsSync.ts`** handles **everything else**: reading/writing local files, HTTP calls with retry logic, OAuth token management, cache invalidation, feature flag gating, and the upload/download orchestration.

### Key Takeaways

1. **Fail-open design**: Sync errors are logged but never block CLI startup or CCR installation — the system degrades gracefully.

2. **Incremental upload**: The interactive CLI only uploads entries that have changed locally (`localEntries[key] !== remoteEntries[key]`), minimizing bandwidth and API load.

3. **Cached fire-and-forget pattern**: Download operations use a shared cached promise (`downloadPromise`) so that simultaneous fire-and-forget calls and awaited calls share a single network request.

4. **500 KB file limit**: Both sides (client and backend) enforce a per-file size cap to prevent large files from being synced.

5. **MD5 checksums**: The API tracks content integrity via MD5 checksums (`UserSyncData.checksum`), though the client currently uses it only in the upload result — the fetch path does not yet validate it.

6. **Feature flag-gated**: Both upload and download paths are controlled by feature flags (`UPLOAD_USER_SETTINGS`, `DOWNLOAD_USER_SETTINGS`) and OAuth scope checks, allowing gradual rollout.

7. **Auth via OAuth**: Sync operations use first-party OAuth tokens with the `user:inference` scope, not `user:profile`, making it compatible with broader CCR use cases.

8. **Internal write suppression**: When writing synced files locally, `markInternalWrite()` is used to prevent the CLI's file-watching change detection from triggering spurious re-sync loops.
