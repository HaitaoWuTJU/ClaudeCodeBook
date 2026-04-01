# Summary of `services/autoDream/`

## Purpose of `autoDream/`

The `autoDream/` directory implements the **autoDream** feature — an automated background memory consolidation system that periodically triggers a forked subagent to review and improve memory files. It acts as a time- and session-gated mechanism to ensure memory stays current without interrupting active development.

## Contents Overview

The directory contains 4 files:

| File | Purpose |
|------|---------|
| `index.ts` | Main orchestrator: entry point, gate logic, forked agent runner, and progress watcher |
| `config.ts` | Lightweight feature flag checker (intentionally minimal to avoid circular deps) |
| `consolidationLock.ts` | File-based locking with mtime-as-timestamp for process coordination |
| `consolidationPrompt.ts` | Multi-phase prompt string builder for the consolidation agent |

## How Files Relate to Each Other

```
                        ┌──────────────────┐
                        │   config.ts      │  (getInitialSettings, getFeatureValue_CACHED)
                        └────────┬─────────┘
                                 │ isAutoDreamEnabled()
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                              index.ts                                    │
│                                                                          │
│  executeAutoDream()  ────────────────────────────────────────────────┐  │
│         │                                                            │  │
│         ▼                                                            │  │
│  initAutoDream()                                                     │  │
│         │                                                            │  │
│         ├── reads: isAutoDreamEnabled() ← config.ts                 │  │
│         │                                                            │  │
│         ├── reads: getOriginalCwd(), getKairosActive(),              │  │
│         │            getIsRemoteMode(), isAutoMemoryEnabled()        │  │
│         │                                                            │  │
│         ├── reads: readLastConsolidatedAt() ────────────────────────┼──┼──► consolidationLock.ts
│         │                                                            │  │
│         ├── reads: listSessionsTouchedSince() ───────────────────────┼──┼───► consolidationLock.ts
│         │                                                            │  │
│         ├── reads/writes: tryAcquireConsolidationLock() ─────────────┼──┼────► consolidationLock.ts
│         │                                                            │  │
│         ├── builds: buildConsolidationPrompt() ─────────────────────┼──┼─────► consolidationPrompt.ts
│         │                                                            │  │
│         └── invokes: runForkedAgent()                                │  │
│                    │                                                 │  │
│                    ├── canUseTool: createAutoMemCanUseTool() ────────┼──┤
│                    ├── onMessage: makeDreamProgressWatcher()         │  │
│                    │   └── updates: DreamTask state                  │  │
│                    └── outputs: memory file writes                   │  │
│                                                                          │
│  On success: completeDreamTask(), appendSystemMessage(),                │
│              rollbackConsolidationLock() on abort                      │
│  On failure: failDreamTask(), rollbackConsolidationLock(priorMtime) ───┼──┘
└──────────────────────────────────────────────────────────────────────────┘
```

**Call sequence for a typical firing:**

1. `executeAutoDream()` is called from `stopHooks`
2. `initAutoDream()` reads the lock file's mtime via `consolidationLock.ts`
3. Time gate: checks hours since mtime ≥ `minHours` (from `config.ts`)
4. Session gate: `consolidationLock.ts` scans session files newer than mtime
5. Lock acquisition: `consolidationLock.ts` verifies PID aliveness
6. Fork: `index.ts` runs `runForkedAgent()` with prompt from `consolidationPrompt.ts`
7. Completion: `consolidationLock.ts` re-stamps mtime; DreamTask is finalized

## Key Takeaways

### Gate Philosophy
Gates are checked in **ascending cost order** — time check (cheapest) before session scan (moderate filesystem I/O) before lock acquisition (filesystem + process check). Each failure short-circuits before expensive operations.

### Dual-Purpose Lock File
The lock file (`.consolidate-lock`) serves two roles simultaneously:

- **mtime** = `lastConsolidatedAt` timestamp for both time-gating and session scanning
- **body** = holding PID for live-process verification

This eliminates a separate "last consolidated" metadata file.

### Scan Throttle Design
When the time gate passes but the session gate fails, the lock mtime is **not updated**. This means:

- The time gate stays open on subsequent calls
- A 10-minute throttle (`lastSessionScanAt`) limits how often sessions are re-scanned
- Once enough sessions accumulate, consolidation fires immediately without waiting for the hour timer

### Rollback Enables Retry
On fork failure, `rollbackConsolidationLock(priorMtime)` rewinds the mtime. Combined with the throttle, this creates a retry-on-next-cycle behavior without requiring manual intervention.

### Intentional Isolation
- `config.ts` is deliberately minimal — it avoids importing forked agent, task registry, or message builders to prevent bundling side-effects when UI components only read the enabled state.
- `consolidationPrompt.ts` is extracted from `dream.ts` so auto-dream can ship independently of KAIROS feature flags.
