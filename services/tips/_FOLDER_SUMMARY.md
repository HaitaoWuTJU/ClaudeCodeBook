# Summary of `services/tips/`

## Purpose of `tips/`

The `tips/` directory implements a complete tip display system for Claude Code CLI that shows contextual "helpful hints" to users (e.g., "Did you know you can use terminal themes?"). The system manages tip selection, scheduling, relevance filtering, and history tracking to ensure tips are shown at appropriate times without being repetitive.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Public API — exports `getTipToShowOnSpinner()` and `getRelevantTips()` |
| `tipRegistry.ts` | Defines ~40 built-in tips with content, cooldown periods, and relevance conditions; also loads custom user tips |
| `tipScheduler.ts` | Core scheduling logic — selects the tip that hasn't been shown in the longest time |
| `tipHistory.ts` | Persistence layer — tracks how many startups have passed since each tip was last shown |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                          Public API                              │
│                    (index.ts exports)                            │
│                                                                 │
│   getTipToShowOnSpinner() ◄──── getRelevantTips()               │
└───────────┬──────────────────────┬──────────────────────────────┘
            │                      │
            ▼                      ▼
┌──────────────────────┐  ┌───────────────────────┐
│    tipScheduler.ts   │  │    tipRegistry.ts     │
│                      │  │                       │
│ • Checks if tips    │  │ • Filters tips by     │
│   enabled            │──►  relevance (platform,│
│ • Selects longest-  │  │   IDE, usage, etc.)   │
│   idle tip          │  │                       │
│ • Records shown     │  │ • Returns Tip[]       │
└─────────┬────────────┘  └───────────────────────┘
          │
          ▼
┌──────────────────────┐     ┌───────────────────────┐
│    tipHistory.ts     │────►│   Global Config       │
│                      │     │  (numStartups,        │
│ • getSessionsSince   │     │   tipsHistory)        │
│   LastShown()        │     └───────────────────────┘
│ • recordTipShown()   │
└──────────────────────┘
```

**Call chain for a tip display:**
1. `index.ts` → `tipScheduler.getTipToShowOnSpinner()`
2. `tipScheduler` → `tipRegistry.getRelevantTips(context)` — filters tips by relevance
3. `tipScheduler` → `tipHistory.getSessionsSinceLastShown(tipId)` — finds longest-idle tip
4. `tipScheduler` → `tipHistory.recordTipShown(tipId)` — logs the display event
5. `tipHistory` → `getGlobalConfig()` / `saveGlobalConfig()` — persists to disk

## Key Takeaways

1. **Contextual Relevance**: Tips are filtered by platform (macOS/Windows/Linux), IDE (VSCode, Cursor, etc.), usage patterns, and feature flags, ensuring users see applicable hints.

2. **Smart Scheduling**: The "longest-idle" algorithm prevents the same tips from appearing repeatedly while prioritizing tips that haven't been shown recently.

3. **Configurable Cooldowns**: Each tip has a `cooldownSessions` period specifying minimum startups between showings, configurable per-tip.

4. **Persistence**: Tip history persists across sessions via `tipsHistory` in global config, tracking `numStartups` as a timestamp proxy.

5. **Analytics Integration**: Tip display events are logged to GrowthBook for A/B testing and usage analysis.

6. **Extensibility**: Custom tips can be added via settings (`spinnerTipsOverride`), allowing users or organizations to define their own hints.
