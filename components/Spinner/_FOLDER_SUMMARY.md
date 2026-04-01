# Summary of `components/Spinner/`

## Purpose of `Spinner/`

Provides a complete, modular spinner/loading indicator system for terminal-based UIs using the Ink library. Supports multiple visual modes (shimmer, flashing, stalled), team collaboration features with multiple concurrent spinners, and accessibility considerations.

## Contents Overview

| File | Role |
|------|------|
| `types.ts` | Shared TypeScript types (`SpinnerMode`, `RGBColor`, `ShimmerIndex`) |
| `constants.ts` | Default spinner character sets and constants |
| `utils.ts` | Color manipulation utilities (interpolation, parsing, HSL→RGB) |
| `index.ts` | Barrel export for public API |
| `Spinner.tsx` | Main orchestrating component with auto/visible/manual modes |
| `SpinnerGlyph.tsx` | Individual spinning glyph with configurable colors |
| `useShimmerAnimation.ts` | Hook calculating shimmer position synced to animation clock |
| `useStalledAnimation.ts` | Hook detecting token stalls with smooth intensity transitions |
| `ShimmerChar.tsx` | Single character with near-highlight shimmer effect |
| `FlashingChar.tsx` | Single character with binary flash between colors |
| `GlimmerMessage.tsx` | Full message with shimmer/glimmer visual effect |
| `TeammateSpinnerLine.tsx` | Single teammate row with status and tokens |
| `TeammateSpinnerTree.tsx` | Container for team leader + teammate spinner rows |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                      Spinner.tsx                         │
│  (orchestrator: modes, visibility, message rendering)   │
└──────────────────────────┬──────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────────┐
│ GlimmerMsg  │    │  useShimmer │    │ useStalledAnim  │
│  .tsx       │    │ .animation  │    │  .ts            │
│             │    │             │    │                 │
│ Uses:       │    │ Uses:       │    │ Uses:           │
│ ShimmerChar │    │ stringWidth  │    │ React useRef    │
│ FlashingChar│    │ useAnimFrame │    │                 │
│             │    │             │    │ Outputs:        │
│ Uses:       │    │ Output:     │    │ isStalled       │
│ interpolate │    │ glimmerIdx  │    │ stalledIntensity│
│ parseRGB    │    │ ref         │    │                 │
└──────┬──────┘    └──────┬──────┘    └────────┬────────┘
       │                  │                    │
       ▼                  ▼                    ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────────┐
│   utils.ts  │◄───│   utils.ts  │    │    Spinner.tsx  │
│             │    │             │    │                 │
│ interpolate │    │ stringWidth │    │ Uses outputs    │
│ toRGBColor  │    │ (ink.js)    │    │ to render       │
│ parseRGB    │    │             │    │ appropriate     │
│ hueToRgb    │    │             │    │ visual mode     │
└─────────────┘    └─────────────┘    └────────┬────────┘
                                               │
                         ┌─────────────────────┼──────────────┐
                         ▼                     ▼              ▼
               ┌───────────────┐    ┌───────────────┐ ┌──────────────┐
               │TeammateSpinner│    │   GlimmerMsg  │ │ SpinnerGlyph │
               │    Tree.tsx   │    │   (continued) │ │              │
               └───────┬───────┘    └───────────────┘ └──────────────┘
                       │
                       ▼
               ┌───────────────┐
               │TeammateSpinner│
               │    Line.tsx   │
               └───────────────┘
```

**Hook Layer**: `useShimmerAnimation` and `useStalledAnimation` provide computed values (indices, intensities) to components.

**Render Layer**: `SpinnerGlyph`, `ShimmerChar`, `FlashingChar`, `GlimmerMessage` consume hook outputs to render styled characters/messages.

**Orchestration Layer**: `Spinner.tsx` and `TeammateSpinnerTree.tsx` coordinate which components render based on mode and state.

**Utility Layer**: `utils.ts` provides pure color and character manipulation functions used throughout.

## Key Takeaways

1. **Animation Sync**: Both animation hooks use a parent-provided `time` clock instead of `Date.now()` or `setInterval`, ensuring effects pause when terminal is blurred.

2. **Color Interpolation**: The system uses RGB-based linear interpolation for smooth color transitions; ANSI themes fall back to binary threshold switching at `opacity > 0.5`.

3. **Character Effects**: Three distinct character effects serve different purposes:
   - **ShimmerChar**: Highlights current + 1 position on each side
   - **FlashingChar**: Uniform binary flash on entire character
   - **GlimmerMessage**: Composes multiple characters with per-segment color control

4. **Stall Detection**: The stalled hook waits 3 seconds after the last token (when no tools are active) before transitioning intensity 0→1 over 2 seconds with smooth lerp interpolation.

5. **Platform Awareness**: `utils.ts` detects Ghostty and macOS to use different spinner characters that render correctly on those platforms.

6. **Tree-Shaking Design**: `index.ts` explicitly excludes "teammate" components from barrel exports; they use dynamic `require()` elsewhere to enable dead code elimination.
