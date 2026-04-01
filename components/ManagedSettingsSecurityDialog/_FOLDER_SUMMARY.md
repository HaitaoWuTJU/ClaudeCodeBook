# Summary of `components/ManagedSettingsSecurityDialog/`

## Purpose of `ManagedSettingsSecurityDialog/`

This directory implements a security-focused confirmation dialog that warns users when their organization has applied managed settings containing potentially dangerous configurations (shell settings, environment variables, or hooks). It ensures users explicitly acknowledge these risks before the settings take effect.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Module export barrel — re-exports the main dialog component and utilities |
| `ManagedSettingsSecurityDialog.tsx` | **Main component** — renders the terminal UI dialog with accept/reject options and keyboard shortcuts |
| `utils.ts` | **Utilities** — extracts dangerous settings from configuration, compares old vs. new settings, formats output for display |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                  External Consumers                      │
│  (settings/managed/ManagedSettingsSetup flow)            │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────────┐
              │    index.ts (barrel)      │
              └────────────┬──────────────┘
                           │ re-exports
            ┌──────────────┴──────────────┐
            ▼                             ▼
┌───────────────────────┐    ┌─────────────────────────┐
│ ManagedSettings       │    │     utils.ts            │
│ SecurityDialog.tsx    │◄───│                         │
│                       │    │ extractDangerousSettings│
│ ┌─────────────────┐   │    │ hasDangerousSettings    │
│ │ PermissionDialog│   │    │ hasDangerousSettings    │
│ │ Select          │   │    │   Changed               │
│ │ Box, Text       │   │    │ formatDangerousSettings │
│ └─────────────────┘   │    │   List                  │
└───────────┬───────────┘    └─────────────────────────┘
            │
            │ uses
            ▼
    ┌───────────────────┐
    │   utils.ts        │ ◄── reads
    │                   │     DANGEROUS_SHELL_SETTINGS
    └─────────┬─────────┘     SAFE_ENV_VARS
              │
              ▼
    ┌───────────────────┐
    │ managedEnv        │
    │ Constants.js      │
    └───────────────────┘
```

**Data flow:**

1. **Input**: `SettingsJson` (managed configuration object) is passed to `extractDangerousSettings()`
2. **Processing**: `utils.ts` filters the input against allowlists (`DANGEROUS_SHELL_SETTINGS`, `SAFE_ENV_VARS`) to identify risky items
3. **Output**: `formatDangerousSettingsList()` produces a displayable list of dangerous setting names
4. **UI**: `ManagedSettingsSecurityDialog.tsx` renders the warning dialog using this formatted list
5. **Decision**: User confirms (triggers `onAccept`) or rejects (triggers `onReject`)

## Key Takeaways

| Aspect | Details |
|--------|---------|
| **Security Model** | Uses an **allowlist** approach for environment variables (`SAFE_ENV_VARS`) — any env var *not* in the whitelist is flagged as dangerous |
| **Hook Detection** | Any non-empty hooks object triggers a dangerous-settings warning |
| **Keyboard Support** | Full keyboard navigation: `↑/↓` for options, `Enter` to confirm, `Esc` to reject, `Ctrl+C` to exit |
| **React Compiler** | The main component uses React compiler optimizations (`_c`, memo cache sentinels) for performance |
| **Change Detection** | `hasDangerousSettingsChanged()` compares old vs. new settings via `jsonStringify` — useful for determining if re-approval is needed |
| **Exit State Hints** | The exit hint changes dynamically based on whether the user has already pressed `Ctrl+C` once (double-tap to exit pattern) |
| **Terminal UI** | Built with `ink` (React for terminals) — uses `Box`, `Text`, `Select`, and custom `PermissionDialog` for consistent styling |
