# Summary of `commands/files/`

## Summary

This module provides a command implementation that lists all files currently tracked in the tool's context, displaying their paths relative to the current working directory.

### Purpose
The `call` function serves as a command handler that retrieves and formats a list of files from the tool's file state cache. It transforms absolute paths into relative paths for better readability.

### Data Flow

```
ToolUseContext.readFileState
        │
        ▼
   cacheKeys() ──► extracts file keys from cache
        │
        ▼
   relative(getCwd(), file) ──► converts absolute → relative paths
        │
        ▼
   LocalCommandResult{ type: 'text', value: string }
```

### Key Implementation Details

| Aspect | Description |
|--------|-------------|
| **Input** | `context.readFileState` (optional file state cache) |
| **Processing** | Extract keys via `cacheKeys()`, transform paths via `relative()` |
| **Output** | Text result with either empty-context message or file list |
| **Unused Parameter** | `_args` (prefixed with underscore by convention) |

### Dependencies

| Module | Purpose |
|--------|---------|
| `path/relative` | Path manipulation |
| `ToolUseContext` | Type definition for tool context |
| `LocalCommandResult` | Return type definition |
| `getCwd` | Retrieve current working directory |
| `cacheKeys` | Extract file keys from state cache |
