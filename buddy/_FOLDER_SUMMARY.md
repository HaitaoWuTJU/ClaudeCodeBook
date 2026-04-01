# Summary of `buddy/`

## Purpose of `buddy/`

The `buddy/` directory implements a **companion creature system** — a collectible pet/avatar feature where users spawn randomized companion creatures with species, cosmetics, and stats. The companion lives alongside the CLI interface as an animated ASCII sprite, responds to name-addressing in the chat, and persists its identity across sessions. It is designed to be a fun, differentiated feature distinct from the core AI chat functionality.

---

## Contents Overview

| File | Role |
|------|------|
| **`types.js`** | All shared constants and TypeScript types — 19 species, 8 hats, 6 eye styles, 5 stats, 5 rarity tiers, weights, and color/star mappings. Also uses a `String.fromCharCode()` trick to avoid embedding literal species names (like "canary") in the bundle. |
| **`companion.js`** | Core generation engine — seeded PRNG (`mulberry32`), string hashing, rarity rolling, stat generation, and the public API (`getCompanion`, `roll`, `rollWithSeed`). Regenerates bones deterministically from `userId` on every read. |
| **`sprites.ts`** | ASCII art data and rendering — 18 species × 3 frames of 5-line sprites, plus 8 hat overlays. Exports `renderSprite`, `renderFace`, `spriteFrameCount`. |
| **`CompanionSprite.tsx`** | Main React UI component — renders the animated sprite with idle/fidget frames, pet-heart burst animations, speech bubbles, and bubble fading. Handles narrow-terminal fallback. |
| **`prompt.ts`** | Prompt/attachment utilities — generates a `companion_intro` attachment for the AI context and instruction text telling the model to respond briefly when addressed. |
| **`useBuddyNotification.tsx`** | Teaser overlay hook — shows a rainbow-colored `/buddy` invitation notification during a 7-day promotional window (April 1–7, 2026). Also exports `findBuddyTriggerPositions` for CLI trigger detection. |

---

## How Files Relate to Each Other

```
buddy/types.js          ← defines Species, Hat, Eye, Rarity, CompanionBones, Companion, etc.
        │
        │  (provides types + constants to everyone)
        ▼
┌──────────────────┐    ┌───────────────────┐
│ companion.js     │    │ sprites.ts        │
│                  │    │                   │
│  - mulberry32()  │    │  - BODIES{}       │
│  - hashString()  │    │  - HAT_LINES{}    │
│  - rollFrom()    │    │  - renderSprite() │
│  - getCompanion()│    │  - renderFace()   │
└────────┬─────────┘    └────────┬──────────┘
         │                       │
         │  (reads config,       │  (reads types)
         │   returns Roll)      │  (returns string[])
         ▼                       ▼
┌──────────────────────────────────────────┐
│         CompanionSprite.tsx              │
│                                          │
│  - Reads: getCompanion() + config        │
│  - Renders: renderSprite() + sprites    │
│  - Manages: tick animation state,       │
│             pet hearts, speech bubbles  │
│  - Computes: companionReservedColumns()  │
└────────────────────┬─────────────────────┘
                     │
         │  (adds companion_intro attachments)
         ▼
┌──────────────────┐    ┌────────────────────────────┐
│ prompt.ts        │    │ useBuddyNotification.tsx   │
│                  │    │                            │
│  - companionIntro│    │  - isBuddyTeaserWindow()   │
│    Text()        │    │  - isBuddyLive()           │
│  - getCompanion  │    │  - RainbowText()            │
│    IntroAttach() │    │  - useBuddyNotification()   │
└──────────────────┘    └────────────────────────────┘
```

**Config persistence contract:**

```
Stored in config.companion:
  StoredCompanion = { companion: { soul, hatchedAt } }

On every read via getCompanion():
  1. Pull soul + hatchedAt from config
  2. Regenerate bones fresh from hash(userId) using companion.js
  3. Merge: { ...soul, hatchedAt, ...bones }
```

This means bones are never persisted — they are recalculated, so species renames or balance changes are always reflected.

---

## Key Takeaways

1. **Deterministic but tamper-resistant.** Every companion's appearance and stats derive from `hashString(userId + SALT)` fed into `mulberry32()`. Users cannot edit config to fake a legendary rarity or shiny — bones regenerate on every read.

2. **Storage is minimal and forward-compatible.** Only the AI-generated `name` + `personality` (`soul`) and a `hatchedAt` timestamp are persisted. This decouples the storage schema from the generation logic entirely.

3. **Caching covers three hot paths.** `rollCache` is a single-entry cache that deduplicates calls from sprite ticks (every 500 ms), PromptInput keystrokes, and turn-change observers — the most frequently called consumers of companion data.

4. **The UI is fully animated.** `CompanionSprite` uses a 500 ms tick loop with 3-frame sprite cycling, a blink frame (`-1`), a pet-heart burst (`PET_HEART_FRAMES = 5`, lasting ~2.5s), and a speech bubble that fades over ~3s after 7s of display.

5. **Narrow terminal handling.** When the terminal is under 100 columns, the sprite collapses to a one-line face + truncated quip rather than breaking layout.

6. **AI instruction separation.** The companion explicitly is *not* the AI — `prompt.ts` instructs the model to keep its companion-addressing responses to one line or less, with the speech bubble carrying the companion's full personality. This keeps chat legible.

7. **Promotional teaser gate.** `useBuddyNotification` blocks the teaser notification until April 1, 2026 (or via the `external === 'ant'` bypass). The `feature('BUDDY')` flag gates the entire system end-to-end.

8. **Bundle obfuscation for internal codenames.** `types.js` uses `String.fromCharCode()` for species names so that a literal string like `"canary"` never appears in the compiled bundle — this evades an excluded-strings check in the build pipeline.
