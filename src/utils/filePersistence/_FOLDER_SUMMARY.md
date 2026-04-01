# Summary of `utils/filePersistence/`

## Purpose of `filePersistence/`

Handles **end-of-turn file persistence** for the Claude Code CLI. When a turn completes, this module detects and uploads files that were modified during the turn, ensuring they are available in subsequent interactions. Supports two deployment modes:

- **BYOC (Bring Your Own Cloud)**: Scans the local `outputs/` directory for modified files and uploads them to the Files API.
- **Cloud/1P mode**: Reserved for future functionality (reading file IDs from extended attributes via rclone sync).

---

## Contents Overview

```
filePersistence/
├── index.ts             # Orchestration layer
├── outputsScanner.ts    # I/O utilities
└── types.ts             # Shared type definitions
```

| File | Role |
|------|------|
| `index.ts` | **Orchestrator** — validates prerequisites, dispatches to handlers, manages analytics, invokes Files API uploads |
| `outputsScanner.ts` | **Scanner** — environment detection, recursive directory listing, file modification detection |
| `types.ts` | **Contracts** — shared interfaces (`PersistedFile`, `FailedPersistence`, `FilesPersistedEventData`) and constants (`FILE_COUNT_LIMIT`, `DEFAULT_UPLOAD_CONCURRENCY`) |

---

## How Files Relate to Each Other

```
┌──────────────────────────────┐
│         types.ts             │  ← shared types + constants (interface only)
└──────────────────────────────┘
              ▲
              │ defines types used by both
              ▼
┌──────────────────────────────┐     ┌──────────────────────────────┐
│       index.ts               │────▶│    outputsScanner.ts         │
│  (orchestration logic)       │     │  (fs operations + detection) │
└──────────────────────────────┘     └──────────────────────────────┘
              │
              │ calls
              ▼
┌──────────────────────────────┐
│    services/api/filesApi.ts  │  ← uploadSessionFiles() upload target
└──────────────────────────────┘
```

**Data flow through the call chain:**

1. `runFilePersistence()` is called at turn end (from `main.ts`)
2. `isFilePersistenceEnabled()` checks prerequisites (feature flag + auth + session ID)
3. `getEnvironmentKind()` determines mode (`byoc` vs `anthropic_cloud`)
4. **BYOC path:**
   - `findModifiedFiles()` → scans `{cwd}/{sessionId}/outputs/`
   - Security filter → removes path-traversal attempts (`..`)
   - Count check → enforces `FILE_COUNT_LIMIT`
   - `uploadSessionFiles()` → parallel upload to Files API
   - `logEvent()` → emits analytics
5. Returns `FilesPersistedEventData` (files + failed) back to caller

---

## Key Takeaways

- **Feature-gated**: Entire module is guarded by `feature('FILE_PERSISTENCE')` and only activates in BYOC environments.
- **BYOC-only implementation**: Cloud mode is a stub; the actual file ID reading from extended attributes is not yet implemented.
- **Security-first scanning**: Explicitly rejects path traversal (`..`) and skips symbolic links to prevent accidental or malicious data exfiltration.
- **Resilient I/O**: Scanner handles race conditions (files deleted between `readdir` and `stat`), missing directories, and parallelizes stat calls with `Promise.all`.
- **Analytics-first design**: Three lifecycle events are logged (`started`, `completed`, `limit_exceeded`) with duration, counts, and error details.
- **Parallel uploads**: Files are uploaded in controlled batches via `DEFAULT_UPLOAD_CONCURRENCY` with per-file error tracking.
