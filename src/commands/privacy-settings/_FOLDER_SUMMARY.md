# Summary of `commands/privacy-settings/`

## Purpose of `privacy-settings/`

This directory implements the **privacy settings command** for a CLI application. It provides users with an interface to view and manage their privacy preferences, including participation in data sharing and the "Grove" feature. The command gates access based on subscriber status and provides a conditional flow depending on whether users have already accepted Grove terms.

---

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command definition with lazy loading, feature gating, and dynamic import |
| `privacy-settings.tsx` | React component implementing the privacy settings dialog flow |

---

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────┐
│                    Privacy Settings Flow                      │
└──────────────────────────────────────────────────────────────┘

index.ts                                          privacy-settings.tsx
─────────────────────────────────────────────────────────────────────────
  Defines command metadata:                          Implements actual UI logic:
  • name: 'privacy-settings'                         • Checks user eligibility (isQualifiedForGrove)
  • description                                      • Fetches Grove settings/config
  • isEnabled() → subscriber check                   • Routes to GroveDialog or PrivacySettingsDialog
  • load() → dynamic import('./privacy-settings')    • Logs analytics events on changes
                                                         • Returns completion message

  Entry point                                         Execution point
  (lazy, registered elsewhere)                        (rendered when command invoked)
```

**Execution Order:**
1. The command loader imports the default export from `index.ts`
2. When user invokes `privacy-settings`, `isEnabled()` validates subscriber status
3. If enabled, `load()` triggers the dynamic import, executing `privacy-settings.tsx`
4. The component fetches data, renders the appropriate dialog, and handles user decisions
5. Analytics events are logged when settings change

---

## Key Takeaways

- **Feature Gating**: Privacy settings are only available to "consumer subscribers" via `isConsumerSubscriber()`
- **Lazy Loading**: The implementation is not bundled upfront—it's loaded on demand to reduce initial bundle size
- **Conditional Dialog Flow**: Users who have already accepted Grove terms see privacy settings directly; others see the Grove terms dialog first
- **Analytics Integration**: Setting changes are tracked via `logEvent()` with specific event names and metadata
- **Graceful Degradation**: If the user is ineligible or API calls fail, a fallback message directs users to the privacy settings URL
- **Concurrent Data Fetching**: Settings and config are fetched in parallel via `Promise.all()` for faster load times
