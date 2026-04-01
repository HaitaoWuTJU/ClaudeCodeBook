# Summary of `commands/upgrade/`

## Purpose of `upgrade/`

Provides a `/upgrade` command for Claude Code that allows users to upgrade their Claude AI subscription to the Max plan (20x rate limits). The command opens a browser to the upgrade page and handles re-authentication post-upgrade.

## Contents Overview

| File | Role |
|------|------|
| `upgrade.ts` | Command metadata definition (name, description, availability, `isEnabled` logic, load function) |
| `upgrade.tsx` | Command implementation (subscription check, browser launch, login flow) |

## How Files Relate to Each Other

```
upgrade.ts (Command Definition)
│
├── type: 'local-jsx' ──────────► Links to upgrade.tsx as component
├── load() ─────────────────────► Dynamic import of upgrade.tsx
│
│   ┌──────────────────────────┐
│   │  When command invoked:   │
│   │                          │
│   │  upgrade.tsx             │
│   │  └─ call(onDone, ctx)    │
│   │     ├─ Check subscription│
│   │     ├─ Open browser      │
│   │     └─ Render <Login>    │
│   └──────────────────────────┘
│
└── Exports: default upgrade command
```

**File Relationship:**
1. `upgrade.ts` acts as the **plugin entry point** — registers the command, declares availability, and lazily loads the implementation
2. `upgrade.tsx` is the **implementation** — contains all business logic (subscription checking, browser opening, UI rendering)

## Key Takeaways

- **Lazy loading**: The heavy `upgrade.tsx` component is only imported when the command runs, keeping initial bundle lean
- **Subscription gating**: The `isEnabled()` check hides the command from enterprise users and when `DISABLE_UPGRADE_COMMAND` env var is set
- **Smart detection**: Checks both cached OAuth tokens and fetches fresh profile data to determine if user already has Max 20x plan
- **Context-only availability**: Only appears in `claude-ai` context, ensuring it never shows outside proper usage
- **Error resilience**: Gracefully handles browser open failures by displaying the URL for manual navigation
- **Re-auth flow**: After upgrade, user must log in again to obtain new API key with elevated limits
