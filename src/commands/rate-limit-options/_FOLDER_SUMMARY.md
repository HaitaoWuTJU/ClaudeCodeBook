# Summary of `commands/rate-limit-options/`

## Summary: `commands/rate-limit-options/`

### Purpose
Implements a hidden, subscriber-only CLI command that presents users with a dialog when they hit their Claude.ai rate limit. The menu offers actionable paths forward—upgrade, add extra usage, or cancel—integrating with billing, subscription, and analytics systems.

### Contents Overview

| File | Role | Key Characteristics |
|------|------|---------------------|
| `rate-limit-options.ts` | Command registry entry | Hidden, local-jsx, lazy-loaded, subscriber-only |
| `rate-limit-options.tsx` | UI implementation | React component, React-compiler output, dialog-based menu |

### How Files Relate to Each Other

```
rate-limit-options.ts (command config)
       │
       ├── imports: isClaudeAISubscriber() from utils/auth
       │
       ├── isEnabled() → checks subscriber status
       │
       └── load() → dynamic import("./rate-limit-options.jsx")
                           │
                           └── rate-limit-options.tsx (actual component)
                                       │
                                       ├── imports: auth, billing, analytics, hooks
                                       │
                                       ├── exports: call() → renders RateLimitOptionsMenu
                                       │
                                       ├── renders: CustomSelect + Dialog components
                                       │
                                       └── user selection → upgrade/extra-usage sub-commands
```

### Key Takeaways

1. **Subscriber-gated**: The command is completely disabled for non-subscribers, hiding both the feature and its existence from non-eligible users.

2. **Dynamic import pattern**: The component is loaded on-demand, keeping the initial bundle small and deferring the expensive UI until the user actually needs it.

3. **Hierarchical options**: The menu adapts its options based on subscription type (`max`, `team`, `enterprise`), billing access, and whether the user is already over their limit.

4. **Sub-command routing**: Selecting "upgrade" or "extra-usage" doesn't navigate away—instead it asynchronously loads and renders the respective sub-command inline, maintaining dialog state.

5. **Feature-flag driven**: Option ordering is controlled by a growthbook feature flag, allowing quick A/B testing of UX without code changes.

6. **Analytics integration**: Every user action (cancel, upgrade click, extra-usage click) is logged for funnel analysis.
