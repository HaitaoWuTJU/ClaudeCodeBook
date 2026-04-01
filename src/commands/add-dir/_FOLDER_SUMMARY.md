# Summary of `commands/add-dir/`

## Purpose of `commands/add-dir/`

Implements the `/add-dir` CLI command, which allows users to add directories to the workspace's allowed working directories. Directories can be added for the current session only or persisted to local settings.

## Contents Overview

| File | Role | Language |
|------|------|----------|
| `index.ts` | Command descriptor & lazy-loader | TypeScript |
| `add-dir.tsx` | React UI & command logic | TypeScript (compiled) |
| `validation.ts` | Directory path validation | TypeScript |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────┐
│  index.ts (command descriptor)                               │
│  - Exports static metadata (name, description, type)        │
│  - Provides load() → lazy-imports add-dir.tsx               │
└──────────────────────────┬───────────────────────────────────┘
                           │ runtime import
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  add-dir.tsx (command implementation)                        │
│  - Renders AddWorkspaceDirectory (UI form)                   │
│  - Calls validateDirectoryForWorkspace(path) ←──────────────┐
│  - Handles result → shows error or confirmation form         ││
│  - On confirm → applies sandbox & permission updates         │
│  - Calls SandboxManager.refreshConfig()                      │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  validation.ts (validation logic)                            │
│  - expandPath() → resolve() → stat()                         │
│  - Checks: exists? isDirectory? not already in working dir?  │
│  - Returns discriminated union: success | pathNotFound | ... │
│  - addDirHelpMessage() generates colored terminal output     │
└──────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Lazy loading**: The heavy React implementation in `add-dir.tsx` is not loaded until the user runs `/add-dir`, keeping startup time fast.
2. **Single syscall validation**: The `stat()` call handles both existence and type checking (directory vs. file) in one filesystem operation.
3. **Type-safe error handling**: `AddDirectoryResult` uses TypeScript discriminated unions so callers can use exhaustive `switch` statements.
4. **Graceful permission errors**: Filesystem `EACCES`/`EPERM` errors are swallowed silently, preventing startup failures from inaccessible configured directories.
5. **Dual persistence paths**: Directories can be session-only (`applyPermissionUpdate`) or persisted to local settings (`persistPermissionUpdate`), with eager config refresh to avoid race conditions.
