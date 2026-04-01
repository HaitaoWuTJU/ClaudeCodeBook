# Summary of `tools/RemoteTriggerTool/`

## Purpose of `RemoteTriggerTool/`

Provides a complete tool implementation for managing scheduled remote Claude Code agents (triggers) via the claude.ai CCR API. The directory implements a self-contained tool with schema validation, API integration, authentication, policy checks, and terminal UI rendering.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Constants-only module exporting tool name, description, and AI prompt instructions |
| `RemoteTriggerTool.ts` | Main tool implementation: schemas, API handlers, auth, policy checks |
| `UI.tsx` | Terminal UI rendering functions using Ink for tool invocation and result display |

## How Files Relate to Each Other

```
index.ts (constants)
    │
    ├── REMOTE_TRIGGER_TOOL_NAME ──→ RemoteTriggerTool.ts (tool name reference)
    ├── DESCRIPTION ──────────────────→ Used for tool registration/display
    └── PROMPT ───────────────────────→ Instructs AI how to use the tool
                                            │
                                            ▼
RemoteTriggerTool.ts (implementation)
    │
    ├── inputSchema ──→ Validates user input (action, trigger_id, body)
    ├── Action handlers ──→ Maps actions to HTTP methods/URLs
    ├── Auth flow ──→ OAuth token management
    └── outputSchema ──→ Formats API responses (status + json)
            │
            ▼
UI.tsx (rendering)
    │
    ├── renderToolUseMessage() ──→ Displays action + trigger_id
    └── renderToolResultMessage() ──→ Displays status code + line count
```

The constants from `index.ts` provide documentation strings and the tool identifier used during registration. `RemoteTriggerTool.ts` contains the actual business logic referenced by those constants. `UI.tsx` provides the presentation layer for terminal display.

## Key Takeaways

- **Self-contained tool**: All logic (validation, API calls, auth, UI) lives within this directory
- **OAuth security**: Authentication is handled in-process with automatic token refresh, avoiding shell exposure
- **Feature-gated**: Requires `tengu_surreal_dali` feature flag and `allow_remote_sessions` policy
- **Five operations**: `list`, `get`, `create`, `update`, `run` — standard CRUD with an explicit run action
- **Deferred, concurrency-safe**: Uses `shouldDefer: true` and `isConcurrencySafe: true` for execution model
- **Terminal-first**: UI uses Ink (not React DOM) for command-line rendering
- **Read-only shortcuts**: `list` and `get` skip policy checks; write operations (`create`, `update`, `run`) require policy validation
- **Error handling**: OAuth and org resolution errors provide descriptive messages to guide user action
