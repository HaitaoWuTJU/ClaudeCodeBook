# Summary of `commands/release-notes/`

## Purpose of `release-notes/`

This directory implements a **local CLI command** that retrieves and displays project release notes/changelog information. The command supports both interactive and non-interactive environments and includes graceful fallback behavior when network access is unavailable.

## Contents Overview

| File | Role |
|------|------|
| `release-notes.ts` | **Command definition** — registers the command with metadata and lazy-loads the implementation |
| `release-notes.js` | **Command implementation** — fetches, caches, formats, and returns changelog data |

## How Files Relate to Each Other

```
release-notes.ts                    release-notes.js
┌─────────────────────────┐         ┌──────────────────────────────┐
│  Export: releaseNotes   │  ──►   │  Implementation: call()      │
│  - description          │         │  - fetchAndStoreChangelog()  │
│  - name                 │         │  - getStoredChangelog()      │
│  - type: 'local'        │         │  - getAllReleaseNotes()       │
│  - supportsNonInteractive│        │  - formatReleaseNotes()       │
│  - load() → dynamic     │         │                              │
└─────────────────────────┘         └──────────────────────────────┘
     ^                                              │
     └────────── imports Command type ──────────────┘
```

**Execution flow:**
1. CLI invokes the command via `releaseNotes.load()`
2. Dynamic import loads `release-notes.js`
3. `call()` executes, fetching/caching changelog, then formats and returns output

## Key Takeaways

- **Lazy loading** — Implementation is loaded on-demand via `import()` to enable code-splitting
- **Fallback chain** — Attempts fresh fetch → cached data → direct URL if all else fails
- **500ms timeout** — Prevents command from hanging on slow/unreachable changelog sources
- **Structured output** — Release notes are formatted with version headers and bullet points
- **Non-interactive support** — Can run in CI/CD environments without user interaction
