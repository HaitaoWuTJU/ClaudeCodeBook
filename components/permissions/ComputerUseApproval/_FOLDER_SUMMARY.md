# Summary of `components/permissions/ComputerUseApproval/`

# `ComputerUseApproval/` Directory Summary

## Purpose

The `ComputerUseApproval/` directory contains the React component responsible for rendering a terminal-based permission approval dialog for Claude Code's "Computer Use" feature. When Claude Code needs to control the user's computer (take screenshots, control the mouse/keyboard, read clipboard, etc.), this dialog presents the user with:

1. **macOS TCC (Transparency, Consent, and Control) permission status** — checking whether the required system-level permissions (Accessibility, Screen Recording) have been granted
2. **App-level allowlist** — showing which specific applications Claude wants to use, which dangerous ("sentinel") categories they belong to, and which system capabilities (clipboard, key combos) are being requested

The user's decisions are compiled into a structured permission response that gates what Claude Code can actually do.

## Contents Overview

| File | Role |
|------|------|
| `index.tsx` | Single React component file containing all the permission UI logic and two sub-panel components (`ComputerUseTccPanel` and `ComputerUseAppListPanel`) |
| `index.test.tsx` | Vitest unit tests covering the permission request/response lifecycle |

### `index.tsx` — Component Hierarchy

```
ComputerUseApproval
├── ComputerUseTccPanel (rendered when tccState is present)
│   ├── Shows Accessibility permission status
│   ├── Shows Screen Recording permission status
│   └── Options: Open Accessibility Settings / Open Screen Recording Settings / Retry
│
└── ComputerUseAppListPanel (rendered when tccState is absent)
    ├── Lists all requested apps with checkboxes
    │   ├── Shows installed vs. not-installed state
    │   ├── Shows "already-granted" state for approved apps
    │   └── Shows sentinel-category warnings (shell, filesystem, system_settings)
    ├── Shows requested grant flags (clipboardRead, clipboardWrite, systemKeyCombos)
    ├── Shows count of apps that will be hidden
    └── Options: Allow All / Deny
```

## How Files Relate to Each Other

```
index.tsx
├── index.test.tsx
│
├── Uses from @ant/computer-use-mcp/types
│   ├── CuPermissionRequest  (input type)
│   ├── CuPermissionResponse (output type)
│   └── DEFAULT_GRANT_FLAGS  (default flag values)
│
├── Uses from @ant/computer-use-mcp/sentinelApps
│   └── getSentinelCategory()  (classifies apps by risk level)
│
├── Uses from ../../design-system/Dialog.js
│   └── Dialog wrapper for terminal rendering
│
├── Uses from ../../CustomSelect/select.js
│   └── Select (dropdown, though mostly checkboxes are used)
│
├── Uses from ../../../utils/
│   ├── execFileNoThrow.js    (opens System Settings URLs)
│   └── stringUtils.js        (plural() helper for UI text)
│
└── Uses from ink.js
    └── Box, Text, etc. for terminal UI primitives
```

The test file (`index.test.tsx`) imports the main component and `DENY_ALL_RESPONSE`, verifying the component renders correctly for various input states (with/without TCC state, with/without sentinel apps).

## Key Takeaways

1. **Two-mode dispatcher pattern**: The component acts as a router — it checks for the presence of `request.tccState`. If TCC permissions are missing, it shows the TCC panel; once those are resolved, the app list panel takes over.

2. **Sentinel app warnings**: The `getSentinelCategory()` function from the MCP package classifies apps into `shell`, `filesystem`, and `system_settings` risk categories. The UI renders these as bold yellow warnings to alert users that approving these apps grants elevated access.

3. **System Settings integration**: On macOS, the component opens System Settings directly via `x-apple.systempreferences:` URLs using `execFileNoThrow`, bypassing the need for AppleScript.

4. **Fine-grained response**: The output `CuPermissionResponse` is not a simple boolean — it separates `granted` (with `bundleId`, `displayName`, `grantedAt`) and `denied` (with reason codes like `user_denied` or `not_installed`) arrays, allowing the MCP server to enforce per-app permissions.

5. **Three grant flags**: Beyond per-app approval, Claude requests three boolean capabilities — `clipboardRead`, `clipboardWrite`, and `systemKeyCombos` — which are presented in a compact summary in the panel.

6. **Installed vs. not-installed detection**: Apps are categorized into three UI states: not-yet-approved (unchecked box), already-granted (dimmed ✓), and not-installed (dimmed ○). Only not-yet-approved apps are selectable.

7. **React Compiler optimization**: The component uses the React compiler runtime (`_c`, `Symbol.for("react.memo_cache_sentinel")`, manual cache arrays `$`) for fine-grained memoization without explicit `useMemo`/`useCallback` hooks.
