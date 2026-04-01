# Summary of `components/StructuredDiff/`

## Purpose of `StructuredDiff/`

This directory provides terminal-based syntax highlighting and structured diff rendering with word-level change visualization. It combines native code (for color computation) with React/Ink components (for terminal UI) to display code differences with precise word-level highlighting.

## Contents Overview

| File | Role |
|------|------|
| `src/color/colorDiffUtils.ts` | Guard layer: checks if `color-diff-napi` is available and provides safe accessors |
| `components/StructuredDiff/Fallback.tsx` | UI component: renders word-level diffs with colored highlighting |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────┐
│     colorDiffUtils.ts (Guard Layer)         │
│  ┌─────────────────────────────────────┐    │
│  │ Check: CLAUDE_CODE_SYNTAX_HIGHLIGHT │    │
│  │ Access: color-diff-napi module      │    │
│  └─────────────────────────────────────┘    │
│                    │                        │
│           (exports functions)               │
│                    ▼                        │
│  ┌─────────────────────────────────────┐    │
│  │    Fallback.tsx (UI Component)      │    │
│  │  ┌─────────────────────────────┐    │    │
│  │  │  Uses theme + color utils   │    │    │
│  │  │  Renders: Box > Text nodes  │    │    │
│  │  └─────────────────────────────┘    │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

1. **`colorDiffUtils.ts`** acts as a gatekeeper—it checks the `CLAUDE_CODE_SYNTAX_HIGHLIGHT` environment variable and only exposes color module functions when enabled
2. **`Fallback.tsx`** consumes these utilities (indirectly via theme system) to render colored diff output with word-level change markers

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Feature toggle** | Word-level syntax highlighting can be disabled via `CLAUDE_CODE_SYNTAX_HIGHLIGHT` env var |
| **Word diff algorithm** | Pairs consecutive remove/add lines, uses `diffWordsWithSpace` for token comparison |
| **Fallback behavior** | When >40% of text changed, reverts to standard line-level diff (not word-level) |
| **Terminal compatibility** | Uses Ink + React for rendering; calculates widths manually for proper wrapping |
| **Native integration** | Relies on `color-diff-napi` (native addon) for color theme processing |
| **Graceful degradation** | Both files handle unavailability gracefully—returning `null` or falling back to simpler rendering |
