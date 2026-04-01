# Summary of `commands/thinkback/`

## Purpose of `thinkback/`

The `thinkback/` directory implements the `/think-back` command for Claude Code, providing users with a "2025 Year in Review" feature. This feature generates personalized ASCII animations celebrating a user's coding journey throughout the year, with options to play, edit, fix, or regenerate the content.

The command is gated behind a Statsig feature flag (`tengu_thinkback`) and lazily loads its implementation, ensuring it only impacts bundle size when the feature is enabled.

---

## Contents Overview

| File | Purpose |
|------|---------|
| `thinkback.ts` | Command definition entry point — registers the command, checks feature gate, and lazily imports the implementation |
| `thinkback.tsx` | Full implementation — handles plugin installation, UI components for installation flow, and animation playback |

### `thinkback.ts` (Entry Point)

A thin orchestration layer that:

- Defines command metadata (`name`, `description`, `type: 'local-jsx'`)
- Checks the `tengu_thinkback` feature gate before enabling
- Defers loading of `thinkback.tsx` until the command is actually invoked

### `thinkback.tsx` (Implementation)

Contains the core logic organized into several concerns:

| Section | Description |
|---------|-------------|
| **Installation Flow** | `ThinkbackInstaller` — handles multi-phase plugin/marketplace installation with user feedback |
| **Menu System** | `ThinkbackMenu` — displays action options (play, edit, fix, regenerate) using Ink components |
| **Playback** | `playAnimation()` — executes animation in terminal with alternate screen mode |
| **AI Prompts** | Template strings for invoking the thinkback skill with different modes |
| **Path Resolution** | `getThinkbackSkillDir()` — locates installed skill files |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                        thinkback.ts                          │
│  (Command registration + feature gate + lazy import)         │
└──────────────────────────┬──────────────────────────────────┘
                           │ lazy-loads when invoked
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       thinkback.tsx                          │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ ThinkbackFlow   │──│ ThinkbackMenu│  │ ThinkbackInstal│  │
│  │ (orchestrator)  │  │ (user input) │  │ ler (wizard)    │  │
│  └────────┬────────┘  └──────────────┘  └────────────────┘  │
│           │                                                    │
│           ▼                                                    │
│  ┌─────────────────┐                                          │
│  │ playAnimation() │                                          │
│  │ (terminal exec) │                                          │
│  └─────────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
         ┌────────────────────────────────────┐
         │        Plugin Ecosystem            │
         │  (marketplace → plugin install)    │
         └────────────────────────────────────┘
```

---

## Key Takeaways

1. **Lazy Loading Strategy** — The command is defined in a small `thinkback.ts` file while the full implementation lives in `thinkback.tsx`. This keeps initial bundle size minimal.

2. **Feature Gating** — Uses `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` to enable/disable the command based on the `tengu_thinkback` gate, allowing gradual rollout.

3. **Multi-Step Installation UX** — The `ThinkbackInstaller` component guides users through installing the marketplace (if needed), the plugin, and enabling it, with clear state feedback.

4. **Terminal Immersion** — Uses Ink's alternate screen mode (`enterAlternateScreen()`/`exitAlternateScreen()`) for a clean, immersive animation playback experience.

5. **Platform-Aware Execution** — Handles cross-platform file opening: `open` (macOS), `start` (Windows), `xdg-open` (Linux).

6. **Skill Integration** — Edit, fix, and regenerate operations invoke the thinkback skill via the Skill tool using mode-specific prompts.

7. **Error Resilience** — Gracefully handles missing skill files (`ENOENT`) without crashing, returning user-friendly feedback.

8. **React Compiler Optimization** — Uses `_c()` cache pattern for optimized component memoization in frequently-rendered components.
