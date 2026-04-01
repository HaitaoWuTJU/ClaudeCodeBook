# Summary of `utils/git/`

## Purpose of `git/`

Provides **filesystem-based Git integration** — reading repo state, configuration, and ignore rules directly from `.git/` files without spawning subprocesses. The directory implements lightweight parsers for Git internals and utilities for managing the global gitignore.

## Contents Overview

| File | Responsibility |
|------|----------------|
| `gitConfigParser.ts` | Parses `.git/config` to extract values by section/subsection/key |
| `gitFilesystem.ts` | Reads Git state (HEAD, refs, worktrees, remotes) from `.git/` files |
| `gitignore.ts` | Checks if paths are ignored; manages `~/.config/git/ignore` |

## How Files Relate to Each Other

```
gitignore.ts
    │
    ├──► uses gitFilesystem.ts  (dirIsInGitRepo, cwd)
    │
    └──► runs `git check-ignore` subprocess
              │
              │
              ▼
gitFilesystem.ts
    │
    ├──► reads .git/HEAD, refs/*, packed-refs, commondir, shallow
    │
    ├──► uses gitConfigParser.ts  (parseGitConfigValue)
    │         │
    │         └──► reads .git/config
    │
    └──► watches .git files + caches branch/head/remoteUrl/defaultBranch
```

**Dependency chain**: `gitignore.ts` → `gitFilesystem.ts` → `gitConfigParser.ts`

All three are independent of each other at the top level but form a hierarchy: gitignore checks repo membership via gitFilesystem; gitFilesystem reads config via gitConfigParser.

## Key Takeaways

- **No `git` subprocess** for reading state — only `git check-ignore` for gitignore queries and `git rev-parse` for initial root detection
- **Security-first design** — ref names and SHAs are validated against strict allowlists to prevent path injection attacks from malicious `.git/HEAD` or ref files
- **Case-sensitivity handling** mirrors Git itself — section/key names are case-insensitive; subsection names are case-sensitive
- **Watch-based caching** in `GitFileWatcher` invalidates computed values when `.git/HEAD`, `.git/config`, or the current branch ref file changes
- **Worktree-aware** — correctly handles `.git` as a file (not directory), `commondir` pointers, and per-worktree HEAD reading
- **Fail-open error handling** — file read errors, non-git directories, and invalid states return `null`/`false` rather than throwing, allowing graceful degradation
