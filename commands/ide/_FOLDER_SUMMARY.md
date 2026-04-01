# Summary of `commands/ide/`

## Purpose of `commands/ide/`

This directory implements the `ide` CLI command for Claude Code, providing terminal-based functionality for:

1. **Detecting** available IDEs on the system (VS Code, Cursor, Windsurf, JetBrains IDEs)
2. **Connecting** to running IDEs for MCP integration
3. **Opening** projects in IDEs via command-line tools
4. **Installing** IDE extensions/plugins automatically
5. **Managing** auto-connect preferences for IDE connections

## Contents Overview

| File | Role | Size |
|------|------|------|
| `index.ts` | Command entry point with lazy loading | ~260 chars |
| `ide.tsx` | Full command implementation | ~400+ lines |

## How Files Relate to Each Other

```
index.ts (entry point)
    │
    └── lazy imports ──► ide.tsx (actual implementation)
                             │
                             ├── UI components (IDEScreen, RunningIDESelector, etc.)
                             ├── IDE detection service (detectIDEs, detectRunningIDEs)
                             ├── MCP client integration (dynamicMcpConfig)
                             └── Analytics/tracking (tengu_ext_ide_command)
```

The **index.ts** acts as a lightweight bootstrap that defers loading the heavy React implementation until the command is actually invoked. This improves initial CLI startup time.

## Key Takeaways

- **Two-stage architecture**: Thin TypeScript type-safe wrapper (`index.ts`) + heavy React/Ink implementation (`ide.tsx`)
- **Multi-IDE support**: VS Code, Cursor, Windsurf, and JetBrains product families
- **MCP integration**: Manages `dynamicMcpConfig` for SSE/WebSocket IDE connections
- **Terminal UI**: Uses Ink (React for CLI) for interactive selection dialogs
- **Worktree-aware**: Respects git worktree sessions for correct project context
- **Analytics tracking**: Logs `tengu_ext_ide_command` events for telemetry
- **Lazy loading pattern**: Common CLI optimization to keep entry points fast
