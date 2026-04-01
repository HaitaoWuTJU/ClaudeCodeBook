# Summary of `commands/cost/`

## Purpose of `cost/`

The `cost/` directory implements a local command that displays the current cost of Claude Code usage. It provides users with visibility into their spending, with different behavior based on subscription status.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Command entry point with lazy-loading and visibility logic |
| `cost.ts` | Core implementation that formats and returns cost information |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────┐
│          commands/cost/index.ts          │
│  (Lazy loader + metadata)                │
│                                         │
│  • Defines Command interface contract    │
│  • Determines visibility (isHidden)      │
│  • Exports `load` that dynamically       │
│    imports cost.ts                       │
└─────────────────┬───────────────────────┘
                  │ dynamic import()
                  ▼
┌─────────────────────────────────────────┐
│          commands/cost/cost.ts           │
│  (Actual implementation)                 │
│                                         │
│  • Checks subscription/overage status    │
│  • Formats cost via formatTotalCost()    │
│  • Returns { type: 'text', value }       │
└─────────────────────────────────────────┘
```

**Data Flow:**

1. User runs `cost` command
2. `index.ts` checks `isClaudeAISubscriber()` and `USER_TYPE` to determine visibility
3. If visible, `load` dynamically imports `cost.ts`
4. `cost.ts` reads `currentLimits.isUsingOverage` and calls `formatTotalCost()`
5. Returns formatted cost message based on user type

## Key Takeaways

- **Lazy loading pattern** keeps startup fast by deferring `cost.ts` import until the command is invoked
- **Conditional visibility** — hidden from regular subscribers, visible to "ant" users
- **Context-aware messaging** — differs between overage users, subscribers, and non-subscribers
- **Clear separation** — metadata/registration logic in `index.ts`, business logic in `cost.ts`
