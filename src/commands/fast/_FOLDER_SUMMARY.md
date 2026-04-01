# Summary of `commands/fast/`

## Purpose of `fast/`

The `fast/` directory implements the Fast Mode toggle functionality for the CLI. Fast Mode is a high-speed research preview feature that provides faster AI responses. This directory contains both the command configuration entry point and the interactive picker dialog UI.

## Contents Overview

| File | Purpose |
|------|---------|
| `commands/fast/index.ts` | Command descriptor with lazy loading — defines how the CLI exposes the `fast` command (description, availability, argument hints) |
| `commands/fast/fast.tsx` | Actual implementation — React component rendering the toggle dialog, plus helper functions for state management and analytics |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────┐
│  User runs: "fast" or "fast on/off"         │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  index.ts (command descriptor)              │
│  - Validates availability (claude-ai/console)│
│  - Checks isFastModeEnabled()               │
│  - Lazy loads ./fast.js                     │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  fast.tsx (implementation)                  │
│  - call(): Entry point, prefetches status   │
│  - FastModePicker: React dialog component  │
│  - applyFastMode(): Updates app state      │
│  - handleFastModeShortcut(): CLI argument   │
│    processing (on/off)                      │
└─────────────────────────────────────────────┘
```

**Flow:**
1. `index.ts` acts as a thin facade — it provides metadata (name, description, availability) and defers all logic to `fast.tsx`
2. When `call()` is invoked, it prefetches fast mode status, determines whether to handle a shortcut argument directly or show the interactive picker
3. The picker renders a dialog with toggle UI, pricing info, and cooldown warnings
4. State changes propagate through `useSetAppState()` and `updateSettingsForSource()`

## Key Takeaways

- **Modular separation**: Command configuration (`index.ts`) is decoupled from implementation (`fast.tsx`), enabling lazy loading and cleaner architecture
- **Dual entry points**: Supports both interactive mode (via picker dialog) and programmatic mode (via `fast on`/`fast off` arguments)
- **Analytics integration**: Tracks both toggles and picker displays for telemetry
- **Cooldown handling**: Gracefully displays unavailability reasons when rate limits are hit
- **React Compiler optimized**: Uses manual cache indexing (`$[0]`, `$[1]`) for performance in the picker component
