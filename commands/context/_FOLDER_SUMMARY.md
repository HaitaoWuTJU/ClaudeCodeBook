# Summary of `commands/context/`

## Summary

## Purpose

The `context/` directory implements the `/context` REPL command that visualizes token consumption and context breakdown—what the AI model sees during API calls. It addresses the common problem where users wonder why their token count doesn't match their message count, by showing the actual processed context after all transformations (compact boundaries, context collapse, micro-compaction).

## Contents Overview

The directory contains four TypeScript/React files with distinct responsibilities:

| File | Type | Responsibility |
|------|------|----------------|
| `index.ts` | Entry Point | Routes to interactive or non-interactive variant based on session type |
| `context.tsx` | Core Logic | Orchestrates the full pipeline: API view transformation → micro-compaction → context analysis → ANSI rendering |
| `context.ts` | Command Handler | Handles slash command, collects data, formats Markdown table output |
| `context-noninteractive.ts` | Fallback | Simple wrapper for non-interactive sessions |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                       User runs /context                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  index.ts                                                       │
│  • getIsNonInteractiveSession() decides:                       │
│    ├─► Interactive  → lazy-loads context.tsx                    │
│    └─► Non-interactive → lazy-loads context.ts                   │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐    ┌─────────────────────────────────┐
│  context.tsx             │    │  context.ts                     │
│  (interactive mode)      │    │  (non-interactive mode)         │
│                         │    │                                 │
│  • toApiView()          │    │  • collectContextData()         │
│  • microcompactMessages()│    │  • formatContextAsMarkdownTable │
│  • analyzeContextUsage() │    │                                 │
│  • ContextVisualization  │    │                                 │
│    (React → ANSI string) │    │                                 │
└─────────────────────────┘    └─────────────────────────────────┘
```

**Pipeline stages (shared logic):**

1. **Compact boundary filter** — `getMessagesAfterCompactBoundary()` removes collapsed spans
2. **Context collapse** — `projectView()` applies collapse transformations (feature-gated)
3. **Micro-compaction** — `microcompactMessages()` further reduces token overhead
4. **Analysis** — `analyzeContextUsage()` calculates token counts per category
5. **Rendering** — Either React component (ANSI colors) or Markdown table

## Key Takeaways

- **Accurate token reporting**: The command shows what the model actually sees, not raw message counts
- **Preprocessing transparency**: Users can verify compact boundaries, collapse rules, and micro-compaction are working as expected
- **Feature-gated context collapse**: Collapse statistics only appear when the `CONTEXT_COLLAPSE` feature is enabled via `bun:bundle`
- **Dual output formats**: Interactive sessions get colored grid visualization; non-interactive sessions get plain Markdown tables
- **Shared data collection**: `collectContextData()` in `context.ts` is the canonical source for context analysis, used by both the TSX renderer and Markdown formatter
