# Summary of `components/LogoV2/`

## Purpose of `LogoV2/`

The `LogoV2/` directory contains the terminal UI components that render the welcome screen, animated decorations, contextual notices, and promotional upsells shown to users at startup or throughout the CLI session. It is the presentation layer of Claude Code's homescreen experience.

## Contents Overview

The directory holds six primary component files:

| File | Role |
|------|------|
| `WelcomeV2.tsx` | Renders the full welcome screen with ASCII-art logo, version info, and theme-aware styling |
| `AnimatedAsterisk.tsx` | Displays a single animated character that sweeps through a rainbow hue cycle and settles to grey |
| `AnimatedClawd.tsx` | A clickable mascot component with frame-based "jump wave" and "look around" animations |
| `ChannelsNotice.tsx` | Conditionally renders one of several notices about channel status (disabled, no auth, policy blocked, or actively listening) |
| `VoiceModeNotice.tsx` | A dismissible notice (shown up to 3×) informing users that `/voice` is available |
| `OverageCreditUpsell.tsx` | An upsell promoting extra usage credits, gated by backend eligibility and capped at 3 impressions |
| `Feed.tsx` | Defines `FeedConfig` and `FeedItem` types for the rotating homescreen feed |
| `Clawd.js` | Static mascot character with named poses (`default`, `look_right`, `look_left`, `crouch`, `crouch_arms_up`) |
| `Opus1mMergeNotice.tsx` | A conflicting notice that suppresses `VoiceModeNotice` when both can't be shown |

Supporting utilities (`index.ts`, `LogoV2.tsx`) assemble and export the composed logo UI, applying feature flags (`KAIROS`, `KAIROS_CHANNELS`, `VOICE_MODE`) and reading `prefersReducedMotion` to gate animations at the top level.

## How Files Relate to Each Other

```
WelcomeV2.tsx  (shell)
├── AnimatedAsterisk   (imported directly or via TeardropLogo)
├── AnimatedClawd      (clickable mascot)
├── ChannelsNotice    (optional notice block)
├── VoiceModeNotice   (optional notice block)
│   └── AnimatedAsterisk
├── OverageCreditUpsell (optional upsell block)
└── Feed             (rotating feed)
    └── OverageCreditUpsell (one of the feed items)
```

**Dependency hierarchy:**

- `AnimatedAsterisk` is the lowest-level animation primitive. It is used directly by `VoiceModeNotice` and indirectly by `TeardropLogo`/`AnimatedClawd`.
- `AnimatedClawd` uses `Clawd` (static poses) and `useAnimationFrame` (shared clock). It respects `prefersReducedMotion` and pauses when scrolled off-screen.
- `ChannelsNotice`, `VoiceModeNotice`, and `OverageCreditUpsell` all share a pattern: read from backend/subscription state → apply a cap or dismiss flag → conditionally render a notice. They are composable and individually feature-flagged.
- `ChannelsNotice` is gated behind `KAIROS` / `KAIROS_CHANNELS` and performs all its reads once at mount time so branches cannot flip mid-session.
- `VoiceModeNotice` and `OverageCreditUpsell` share an impression-counting mechanism via `getGlobalConfig`/`saveGlobalConfig`, both capping at 3 shows.
- `OverageCreditUpsell` doubles as a `FeedItem` via `createOverageCreditFeed()` for the rotating homescreen feed.

## Key Takeaways

1. **Feature flags gate the entire tree.** `LogoV2.tsx` conditionally requires `ChannelsNotice` and `VoiceModeNotice` behind `KAIROS`/`KAIROS_CHANNELS` and `VOICE_MODE`. These files should never be statically imported from unguarded code — the bundle omits them entirely when flags are `false`.

2. **Animation is opt-out by default.** Every animated component (`AnimatedAsterisk`, `AnimatedClawd`, the feed rotator) checks `prefersReducedMotion`. When `true`, animations are skipped or the component renders in its final settled state.

3. **State is captured once at mount.** `ChannelsNotice`, `VoiceModeNotice`, and `OverageCreditUpsell` all perform their reads via `useState(() => ...)` initializers or one-time `useEffect` reads. This prevents render-time state changes from flipping conditional branches mid-session.

4. **React Compiler is enabled for the entire directory.** All components use `react/compiler-runtime` (`_c`, `Symbol.for("react.memo_cache_sentinel")`). This emits fixed cache slot arrays (`$[n]`) instead of `React.memo` / `useMemo` / `useCallback`. The compiler also injects bailout checks for early `return null` paths.

5. **Ink.js is the rendering layer.** All components use `Box`, `Text`, and `useTheme` / `useAnimationFrame` from `ink.js` — a React-compatible virtual DOM for terminal environments. No DOM APIs are used directly.

6. **Global config is the persistence layer** for dismissals and impression counts. It survives CLI restarts but is local to the user's machine (not backend-synced).

7. **Viewport-pause prevents flicker.** `AnimatedClawd` and `AnimatedAsterisk` pass refs to `useAnimationFrame`, which pauses the shared animation clock when the row enters scrollback. This avoids stale renders on fast terminal interactions.
