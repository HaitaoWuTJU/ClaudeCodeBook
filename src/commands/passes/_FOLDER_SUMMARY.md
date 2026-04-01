# Summary of `commands/passes/`

## Summary: `passes/`

### Purpose

The `passes/` directory implements a CLI command that presents a referral program interface, allowing users to share free weeks of Claude Code with friends and potentially earn additional usage rewards. The system tracks referral eligibility and manages first-visit state to control upsell messaging.

### Contents Overview

| File | Type | Role |
|------|------|------|
| `passes.tsx` | Command Definition | Exports the command configuration object with lazy-loading, visibility logic, and dynamic description |
| `passes.ts` | Command Implementation | Handles first-visit tracking, analytics logging, and renders the `<Passes>` UI component |

### How Files Relate to Each Other

```
passes.tsx                              passes.ts
────────────                            ─────────
┌─────────────────────────┐            ┌──────────────────────────────┐
│  Command Definition     │            │  Command Implementation     │
│  ─────────────────      │   loads    │  ────────────────────────   │
│  • type: 'local-jsx'    │ ─────────> │  • First-visit detection    │
│  • name: 'passes'       │            │  • Analytics logging        │
│  • isHidden getter      │  import()  │  • Config persistence        │
│  • description getter   │            │  • Renders <Passes>         │
│  • load() function      │            │                              │
└─────────────────────────┘            └──────────────────────────────┘
```

1. When a user invokes the `passes` command, the lazy-loaded `passes.ts` module executes
2. `passes.ts` handles business logic (first-visit tracking, analytics) and returns the `<Passes>` React component from `components/Passes/Passes.js`
3. Both files access shared services in `services/api/referral.js` to determine eligibility and reward status

### Key Takeaways

- **Lazy Loading**: The command uses dynamic import (`import('./passes.js')`) to defer loading the implementation until the command is actually invoked, optimizing initial bundle size
- **Eligibility-Based Visibility**: The command is hidden unless the user is both eligible for the referral program and has cached eligibility data
- **First-Visit Tracking**: The system persists a `hasVisitedPasses` flag in global config to suppress upsell modals after the initial visit
- **Analytics Integration**: Every pass page visit triggers a `tengu_guest_passes_visited` event with `is_first_visit` metadata for tracking user engagement
- **Dynamic Content**: The command description changes based on whether the user has an active referrer reward, personalizing the messaging
- **Command Interface Compliance**: Both files use TypeScript's `satisfies` operator and type imports to ensure strict adherence to the `Command` interface contract
