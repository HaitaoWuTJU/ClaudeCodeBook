# Summary of `components/TrustDialog/`

## Purpose of `TrustDialog/`

The `TrustDialog/` directory implements a security gate mechanism for the CLI. Before granting file system access to a workspace, the component performs a comprehensive audit of potentially dangerous configurations and presents a user-facing confirmation dialog requiring explicit acknowledgment.

The core purpose is **informed consent**: ensuring users are aware of elevated permissions in their workspace before the tool proceeds with operations that could modify files, execute shell commands, or interact with cloud providers.

## Contents Overview

| File | Purpose |
|------|---------|
| `TrustDialog.tsx` | Main React component rendering the permission dialog; orchestrates trust checks, handles user input, persists acceptance state, and logs analytics |
| `utils.ts` | Pure utility functions that scan project and local settings files to identify which configurations contain trust-relevant settings |

## How Files Relate to Each Other

```
TrustDialog.tsx
    │
    ├── imports utils.ts
    │
    ├──┌─────────────────────────────────────────┐
    │  Uses helper functions from utils.ts:      │
    │                                          │
    │  • getHooksSources()                      │
    │  • getBashPermissionSources()            │
    │  • getOtelHeadersHelperSources()         │
    │  • getApiKeyHelperSources()              │
    │  • getAwsCommandsSources()               │
    │  • getGcpCommandsSources()               │
    │  • getDangerousEnvVarsSources()          │
    │                                          │
    │  ┌─────────────────────────────────────┐  │
    │  │       data flow                     │  │
    │  │                                     │  │
    │  │  Project/local settings files       │  │
    │  │          ↓                         │  │
    │  │  utils.ts: scan & extract paths     │  │
    │  │          ↓                         │  │
    │  │  TrustDialog.tsx: aggregate results │
    │  │          ↓                         │  │
    │  │  Render dialog with findings        │  │
    │  └─────────────────────────────────────┘  │
    └─────────────────────────────────────────┘
```

**Division of responsibility:**

- **utils.ts** is purely analytical — it reads settings files and returns arrays of file paths containing dangerous configurations. It contains no UI logic or side effects.
- **TrustDialog.tsx** handles presentation, user interaction, state persistence, and analytics. It consumes the utilities' output and drives the trust workflow.

## Key Takeaways

1. **Layered security check**: The dialog scans **7 distinct categories** of potentially dangerous configurations — MCP servers, hooks, bash permissions, OpenTelemetry helpers, API key helpers, cloud provider commands (AWS/GCP), and environment variables.

2. **Two-tier persistence**: Trust acceptance is stored differently based on scope:
   - **Home directory** (`~/`): Session-only (memory) via `setSessionTrustAccepted`
   - **Project directories**: Persisted to `.claude/settings.json` for long-term trust

3. **Settings file abstraction**: The utility layer abstracts away the distinction between `projectSettings` and `localSettings`, returning paths like `'.claude/settings.json'` so the UI can display which config files contain dangerous settings.

4. **Analytics-first design**: Every dialog invocation and acceptance is logged with granular metadata (which sources were detected), enabling security audits and usage tracking.

5. **React Compiler optimized**: `TrustDialog.tsx` uses React compiler patterns (`_c` symbols, memo cache sentinels) for render performance in terminal environments.
