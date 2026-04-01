# Summary of `commands/compact/`

## Purpose of `compact/`

Implements the `/compact` command for compressing conversation history to reduce token usage. Provides multiple compaction strategies (session memory, reactive, and traditional summarization) with cleanup, caching, and user feedback.

## Contents Overview

```
commands/compact/
├── index.ts      # Command plugin entry point (lazy-loads actual logic)
└── compact.ts    # Core compaction implementation
```

| File | Role |
|------|------|
| `index.ts` | Command registry definition; defines metadata (name, description, argument hints) and lazy-loads `compact.ts` when invoked |
| `compact.ts` | Command logic; orchestrates compaction strategies, handles errors, manages cache state, and returns results with display text |

## How Files Relate to Each Other

```
index.ts (entry point)
  │
  └─► lazy imports ─► compact.ts (implementation)
                            │
                            ├─► getMessagesAfterCompactBoundary()  ─► filters REPL snipped messages
                            ├─► trySessionMemoryCompaction()        ─► quick session-memory compaction (if no custom instructions)
                            ├─► compactViaReactive()               ─► reactive-only mode compaction (if REACTIVE_COMPACT enabled)
                            └─► microCompact + compactConversation ─► traditional summarization (fallback)
```

- `index.ts` acts as a thin plugin wrapper that defers all heavy logic to `compact.ts`
- `compact.ts` owns the full data pipeline: message filtering → strategy selection → execution → cleanup → result formatting
- Both files share the `Command` interface contract from `../../commands.js`

## Key Takeaways

- **Multiple fallback strategies**: Compaction attempts session memory first, then reactive mode, then traditional summarization — allowing graceful degradation
- **Feature-gated capabilities**: `REACTIVE_COMPACT` and `PROMPT_CACHE_BREAK_DETECTION` flags control optional reactive compaction and cache safety notifications
- **Custom summarization**: Users can provide custom instructions to guide how messages are summarized instead of using the default behavior
- **State cleanup**: Always runs post-compaction cleanup (`suppressCompactWarning`, `getUserContext.cache.clear`, `runPostCompactCleanup`) to keep the session consistent
- **Non-interactive support**: Fully compatible with CI environments (`supportsNonInteractive: true`)
- **Disabled via env**: Entire command can be toggled off with `DISABLE_COMPACT` environment variable
