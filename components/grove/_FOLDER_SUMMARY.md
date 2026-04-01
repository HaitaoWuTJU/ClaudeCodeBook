# Summary of `components/grove/`

## Purpose of `grove/`

This directory contains the **Grove** feature — a terminal/CLI UI layer for managing Anthropic's policy and terms update dialogs. It implements interactive consent flows that allow users to review, accept, or decline data usage ("Help improve Claude") with different content experiences based on a **grace period deadline** (October 8, 2025).

The components render in terminal environments using **Ink** (React for CLIs) rather than traditional web browsers.

---

## Contents Overview

| File/Component | Purpose |
|----------------|---------|
| `GroveDialog` | Main entry point; checks whether grove notice should be shown, renders the appropriate dialog with policy content and selectable options |
| `PrivacySettingsDialog` | Standalone settings dialog for users to review and toggle their "Help improve Claude" preference |
| `GracePeriodContentBody` | Content displayed during the pre-deadline period explaining policy changes and providing Accept/Defer/Decline options |
| `PostGracePeriodContentBody` | Simplified content shown after October 8, 2025 with Accept/Decline options only (defer removed) |
| `NEW_TERMS_ASCII` | ASCII art banner displayed in dialog headers |
| `GroveDecision` type | Union of possible user choices: `accept_opt_in`, `accept_opt_out`, `defer`, `escape`, `skip_rendering` |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                    Entry Points                             │
├──────────────────────────┬──────────────────────────────────┤
│  GroveDialog             │  PrivacySettingsDialog            │
│  (integrated dialog)     │  (standalone settings)            │
├──────────────────────────┴──────────────────────────────────┤
│                          │                                   │
│         ┌────────────────┴────────────────┐                  │
│         │     Shared Content Bodies      │                  │
│         ├───────────────────────────────┤                  │
│         │ GracePeriodContentBody         │ ← Before Oct 8   │
│         │ PostGracePeriodContentBody     │ ← After Oct 8    │
│         └───────────────────────────────┘                  │
│                          │                                   │
│         ┌────────────────┴────────────────┐                  │
│         │        Data Layer (API)          │                  │
│         ├─────────────────────────────────┤                  │
│         │ getGroveSettings()              │                  │
│         │ getGroveNoticeConfig()           │                  │
│         │ calculateShouldShowGrove()       │                  │
│         │ updateGroveSettings()            │                  │
│         │ markGroveNoticeViewed()          │                  │
│         └─────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

**GroveDialog** serves as the integrated policy update modal, while **PrivacySettingsDialog** provides standalone access to the privacy controls. Both delegate rendering to either **GracePeriodContentBody** or **PostGracePeriodContentBody** based on the current date relative to the deadline. All components interact with the same underlying grove settings and analytics services.

---

## Key Takeaways

- **Dual Entry Points**: Grove can appear as an integrated policy update modal (`GroveDialog`) or as a standalone settings screen (`PrivacySettingsDialog`)
- **Time-Based Content Switching**: Content automatically switches from grace period to post-deadline based on the October 8, 2025 cutoff date
- **Analytics Integration**: Every user interaction (view, accept, decline, defer, escape) is tracked via dedicated analytics events
- **Domain Exclusion Handling**: Users in excluded domains can only opt-out (data usage cannot be enabled for their domain)
- **Data Retention Implications**: 
  - Opt-in → 5-year data retention
  - Opt-out → 30-day retention (system default)
- **Keyboard-First UX**: Full keyboard navigation support with dynamic shortcut hints based on the current dialog state
- **React Compiler Optimized**: Uses manual memoization (`_c` from `react/compiler-runtime`) for performance in terminal rendering environments
