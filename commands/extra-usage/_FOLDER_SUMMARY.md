# Summary of `commands/extra-usage/`

## Summary

### Purpose of `extra-usage/`

Provides a `/extra-usage` command implementation that allows users to manage additional usage limits when they hit their standard quotas. The feature serves two distinct user experiences:

1. **Interactive users** → Renders a React component that may trigger a login flow
2. **Non-interactive/script users** → Returns plain text messages with guidance

### Contents Overview

| File | Role |
|------|------|
| `index.ts` | Entry point; exports two command variants (interactive & non-interactive) with lazy loading and feature flag checks |
| `extra-usage-core.ts` | Core business logic; fetches utilization, checks eligibility, creates admin requests, or opens browser |
| `extra-usage.tsx` | Interactive command handler; renders React `Login` component when authentication is needed |
| `extra-usage-noninteractive.ts` | Non-interactive handler; formats core results as user-friendly text messages |

### How Files Relate to Each Other

```
index.ts (export)
    │
    ├─► extra-usage.tsx (interactive)
    │       │
    │       └─► extra-usage-core.ts
    │
    └─► extra-usage-noninteractive.ts (non-interactive)
            │
            └─► extra-usage-core.ts
```

1. **`index.ts`** acts as the router, exposing both command variants based on session type and `DISABLE_EXTRA_USAGE_COMMAND` env var.

2. Both **`extra-usage.tsx`** and **`extra-usage-noninteractive.ts`** delegate to **`extra-usage-core.ts`** for the actual logic:
   - Core returns a discriminated union: `{ type: 'message', value: string }` or `{ type: 'browser-opened', url, opened }`
   - Interactive handler renders JSX or passes message to callback
   - Non-interactive handler transforms result into plain text

3. **`extra-usage-core.ts`** is the single source of truth for eligibility checks, admin request creation, and browser URL selection.

### Key Takeaways

- **Feature toggle**: Set `DISABLE_EXTRA_USAGE_COMMAND=1` to globally disable the command
- **Session-aware**: Automatically chooses between interactive (JSX) and non-interactive (text) variants
- **Error resilience**: Core logic catches errors and falls through gracefully rather than failing
- **Unified backend**: Both UI variants share the same business logic, ensuring consistent behavior regardless of session type
- **Lazy imports**: Command implementations are dynamically imported to avoid loading unused code paths
