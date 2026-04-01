# Summary of `services/teamMemorySync/`

## Purpose of `services/teamMemorySync/`

Implements a secure, bidirectional sync service for **team memory** — shared key-value file storage scoped to a GitHub repository that is accessible to all organization members authenticated via first-party Anthropic OAuth. The directory provides:

1. **Bidirectional sync** between local filesystem (`~/.claude/teams/<org>/<repo>/`) and a remote API
2. **Client-side secret scanning** to prevent credentials from leaving the user's machine
3. **Write guarding** integrated into file tooling to block secret writes to team memory paths
4. **File watching** with debounced push to the server whenever local files change
5. **Conflict resolution** via HTTP 412 + hash probe for optimistic locking
6. **Structured error handling** including 413 (quota exceeded) with per-org limits
7. **Analytics telemetry** for sync operations and secret detection events

---

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Core sync engine: `pullTeamMemory`, `pushTeamMemory`, `syncTeamMemory`, `fetchTeamMemory`, `readLocalTeamMemory`, `uploadTeamMemory`, `batchDeltaByBytes`, `SyncState` management |
| `secretScanner.ts` | Curated regex patterns for 30+ secret types (AWS, GCP, Azure, AI providers, VCS tokens, messaging APIs, etc.) — runs client-side only |
| `teamMemSecretGuard.ts` | Security guard `checkTeamMemSecrets(filePath, content)` called by file tooling to block secret writes before they persist |
| `types.ts` | Zod schemas (`TeamMemoryDataSchema`, `TeamMemoryTooManyEntriesSchema`) and TypeScript types for the API contract |
| `watcher.ts` | `fs.watch` wrapper with debounced push, initial pull on startup, and graceful shutdown |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONFIGURED BINARY                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                  ┌───────────┴────────────┐
                  ▼                         ▼
┌─────────────────────────┐   ┌─────────────────────────────┐
│   teamMemSecretGuard.ts  │   │         watcher.ts          │
│  (file write protection) │   │  (debounced push + watching)│
│                         │   └──────────────┬──────────────┘
│ checkTeamMemSecrets()   │                  │ notifies
│   ↓                      │                  ▼
│ scanForSecrets()         │   ┌─────────────────────────────┐
└─────────────────────────┘   │        secretScanner.ts      │
                              │   (regex patterns for 30+     │
                              │    secret types)              │
                              └──────────────┬──────────────┘
                                             │ used by
                                             ▼
                              ┌─────────────────────────────┐
                              │         index.ts             │
                              │  (pull, push, delta, retry,  │
                              │   batch, conflict resolution)│
                              └──────────────┬──────────────┘
                                             │ reads/writes
                                             ▼
                              ┌─────────────────────────────┐
                              │         types.ts             │
                              │  (Zod schemas + TypeScript   │
                              │   types for API contract)   │
                              └─────────────────────────────┘
```

**Data flow, end to end:**

1. **Startup** → `watcher.ts` calls `pullTeamMemory()` to fetch and write server entries locally
2. **User edits** → Tools write to team memory files; `teamMemSecretGuard.ts` scans content via `secretScanner.ts` and blocks secret writes
3. **File change detected** → `watcher.ts` debounces 2s, then calls `pushTeamMemory()` in `index.ts`
4. **Push** → `index.ts` reads local files, diffs checksums against `serverChecksums`, batches delta into ≤200KB chunks, uploads via PUT
5. **Conflict** → On 412, `index.ts` probes `?view=hashes`, refreshes state, retries delta (up to 2×)
6. **Telemetry** → All operations emit analytics events via `logEvent`

---

## Key Takeaways

### Security Model
- **Secrets never leave the machine** — `secretScanner.ts` runs entirely client-side; `teamMemSecretGuard.ts` intercepts writes before they persist. The server never sees credentials.
- **Secret values are not logged** — `scanForSecrets` returns only rule IDs/labels; `checkTeamMemSecrets` returns a descriptive error message referencing matched *types*, not actual secrets.
- **Feature-flagged guard** — `checkTeamMemSecrets` is designed to be called unconditionally by tooling; the `TEAMMEM` feature flag makes it a no-op when not configured.

### Sync Semantics
| Direction | Strategy | Conflict Handling |
|-----------|----------|-------------------|
| Pull (server → local) | Server wins | Overwrites local with server content |
| Push (local → server) | Local wins (optimistic) | 412 → hash probe → retry with fresh server state (≤2×) |

- **No deletions propagate** — deleting a local file does not remove it from the server; the next pull restores it.
- **Permanent failures are suppressed** — `no_oauth`, `no_repo`, and non-retryable 4xx errors (except 409/429) halt retries immediately to prevent unbounded retry storms.

### API Design
- **Repo-scoped** — identified by `getGithubRepo()` hash; only github.com remotes are supported
- **ETag-based caching** — server returns `checksum` as ETag; clients send `If-None-Match` for 304 responses
- **Delta uploads** — only changed keys are PUT; unchanged keys are preserved server-side
- **Batched bodies** — greedy bin-packing into ≤200KB chunks to satisfy API gateway limits
- **Structured 413 errors** — server returns `max_entries` and `received_entries` in the error body; these are cached to avoid repeated 413s

### Operational Concerns
- **Watcher fd management** — uses `fs.watch({ recursive: true })` instead of chokidar to avoid fd exhaustion; macOS FSEvents uses O(1) fds regardless of file count
- **Bootstrap protection** — watcher starts on empty directories to avoid dead zones where files written before the first tool-use event are never synced
- **Debounce** — 2-second window prevents rapid-fire pushes when multiple files are written in quick succession
- **Graceful shutdown** — `stopTeamMemoryWatcher` awaits in-flight pushes and flushes pending changes before exiting
