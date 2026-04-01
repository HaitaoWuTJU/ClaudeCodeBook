# Summary of `commands/usage/`

## Purpose of `usage/`

This directory implements a **Settings → Usage tab** command that displays the user's current plan usage limits. It provides a UI component for users to view their API usage, including conversation counts, message limits, and storage quotas.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command registration entry point — exports metadata, defines availability |
| `usage.tsx` | Command implementation — renders `<Settings defaultTab="Usage" />` |

**Related components (in `components/Settings/`):**

| File | Role |
|------|------|
| `Settings.js` | Container component with tab navigation |
| `Usage.js` | Tab content displaying usage limits and progress bars |
| `UsageRow.js` | Reusable row component for individual metrics |
| `index.js` | Module entry point |

**Data source (in `lib/`):**

| File | Role |
|------|------|
| `plan.js` | Plan type definitions and limit constants |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│  User invokes "usage" command                               │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  commands/usage/index.ts                                   │
│  • Validates availability (claude-ai only)                  │
│  • Dynamically imports usage.tsx                            │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  commands/usage/usage.tsx                                   │
│  • Renders <Settings defaultTab="Usage">                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  components/Settings/Settings.js                           │
│  • Tabs: General | Usage | Account                          │
│  • Renders Usage.js tab by default                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  components/Settings/Usage.js + UsageRow.js                 │
│  • Displays plan limits from lib/plan.js                    │
│  • Shows progress bars and usage stats                      │
└─────────────────────────────────────────────────────────────┘
```

## Key Takeaways

- **Modular architecture:** The command entry point (`index.ts`) separates metadata/availability from implementation (`usage.tsx`), enabling lazy loading
- **Tab-based UI:** Reuses the existing Settings infrastructure with a focused default tab
- **Reusable components:** `UsageRow` promotes consistency across different usage metrics
- **Plan-driven data:** Usage limits come from centralized `lib/plan.js` constants
- **Limited availability:** Only accessible in the `claude-ai` environment
