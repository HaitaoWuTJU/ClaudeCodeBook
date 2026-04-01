# Summary of `commands/permissions/`

## Purpose of `commands/permissions/`

This directory implements a **permissions management command** for a tool-calling or AI agent system. It allows users to configure which tools are allowed or denied by managing permission rules.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Entry point; exports a static command configuration with lazy-loaded implementation |
| `permissions.tsx` | Actual command implementation; renders UI and handles retry logic |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────┐
│  index.ts (lazy entry point)                │
│  ├── type: 'local-jsx'                      │
│  ├── name: 'permissions'                    │
│  ├── aliases: ['allowed-tools']             │
│  └── load() → dynamic import                │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│  permissions.tsx (actual implementation)    │
│  ├── Renders <PermissionRuleList />          │
│  ├── Handles onExit callback                │
│  └── Manages onRetryDenials via messages    │
└─────────────────────────────────────────────┘
```

**Flow**: `index.ts` declares command metadata and defers loading → when invoked, `permissions.tsx` loads on-demand, renders the permission rule management UI, and can queue retry messages for denied commands.

## Key Takeaways

1. **Lazy loading pattern**: The command is split into a thin index and a heavy implementation to optimize initial load time
2. **Local JSX command type**: Uses the `'local-jsx'` command type, indicating a locally-rendered React component
3. **Two entry points**: The command can be triggered via `permissions` or `allowed-tools`
4. **Retry mechanism**: When permission denials occur, a retry message is queued for later processing rather than immediate execution
5. **Separation of concerns**: Configuration metadata lives separately from UI/rendering logic, making the command discoverable without loading React runtime
