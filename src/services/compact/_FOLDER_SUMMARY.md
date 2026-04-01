# Summary of `services/compact/`

## Purpose of `compact/`

This directory implements the **context window compaction system** for the CLI assistant, responsible for preventing `prompt_too_long` errors by summarizing and condensing long conversation histories. The system operates on two levels:

| Level | Mechanism | Files |
|-------|-----------|-------|
| **Lightweight** | Microcompact — selective clearing of tool results/uses before each API call | `apiMicrocompact.ts`, `timeBasedMCConfig.ts` |
| **Heavyweight** | Full compaction — LLM-powered summarization of conversation history | `autoCompact.ts`, `compact.ts`, `sessionMemoryCompact.ts`, `postCompactCleanup.ts` |

## Contents Overview

```
compact/
├── apiMicrocompact.ts         # Strategy definitions for per-call content clearing
├── autoCompact.ts             # Orchestrator: decides when to compact, routes to appropriate engine
├── compact.ts                 # Core compaction engine: LLM summarization, message transformation
├── postCompactCleanup.ts      # State reset after compaction (flags, hooks, session memory)
├── sessionMemoryCompact.ts    # Experimental LLM-free compaction using session memory file
└── timeBasedMCConfig.ts       # Remote config loader for time-based microcompact thresholds
```

| File | Role | Lines |
|------|------|-------|
| `autoCompact.ts` | Orchestrator / decision engine | ~500 |
| `compact.ts` | Core summarization engine (LLM call, message transformation) | ~1,800 |
| `sessionMemoryCompact.ts` | Experimental lighter compaction via session memory | ~650 |
| `postCompactCleanup.ts` | Post-compaction side-effect cleanup | ~300 |
| `apiMicrocompact.ts` | Strategy definitions for microcompact | ~150 |
| `timeBasedMCConfig.ts` | Remote config for time-based triggers | ~40 |

## How Files Relate to Each Other

```
Before each API call (microcompact — lightweight):
┌─────────────────────────────────────────────────────────────┐
│  microcompactMessages()                                     │
│  ├── Reads: timeBasedMCConfig.ts (gap threshold)            │
│  └── apiMicrocompact.ts (strategy: what to clear)          │
│        • Clears old tool results (shell, grep, glob…)      │
│        • Preserves editing tools (file edit/write)          │
│        • Optionally preserves/thins thinking blocks         │
└─────────────────────────────────────────────────────────────┘

Proactive auto-compact (heavyweight — triggered by thresholds):
┌─────────────────────────────────────────────────────────────┐
│  autoCompactIfNeeded()                                      │
│  ├── Checks: isAutoCompactEnabled()?                        │
│  ├── Checks: shouldAutoCompact()                            │
│  │     ├── respect circuit breaker (3 consecutive failures)│
│  │     ├── respect recursion guards (compact/session agents)│
│  │     └── respect feature flags (CONTEXT_COLLAPSE, etc.)   │
│  └── Routes:                                                │
│       ├── trySessionMemoryCompaction() ──────────────────┐  │
│       │   └── sessionMemoryCompact.ts                    │  │
│       │       ├── Reads session memory file               │  │
│       │       ├── Computes keep-index with invariants     │  │
│       │       ├── Falls back to legacy if no memory/stale │  │
│       │       └── Triggers hooks (CLAUDE.md restoration)  │  │
│       └── compactConversation() ───────────────────────┐  │
│           └── compact.ts                                 │  │
│               ├── Runs pre-compact hooks                  │  │
│               ├── Calls LLM to summarize                  │  │
│               ├── Streams + retries on PTL                │  │
│               ├── Restores files/skills/plans/async state │  │
│               ├── Annotates boundary marker               │  │
│               └── Returns CompactionResult                │  │
│                                                             │
│  After either engine:                                       │
│  └── runPostCompactCleanup() ← postCompactCleanup.ts       │
│        ├── Resets compaction flags                         │
│        ├── Clears post-compact markers                      │
│        ├── Resets agent mode/flags                         │
│        ├── Handles session memory file                     │
│        └── Resets transcript state                          │
└─────────────────────────────────────────────────────────────┘
```

**Critical invariant**: `compact.ts` never mutates the original message array in memory. It returns a transformed copy, maintaining the invariant that all message mutations are derived from previous states.

## Key Takeaways

### Token Threshold Architecture

| Threshold | Value | Purpose |
|-----------|-------|---------|
| Warning | context_window − 20K | UI warning to user |
| Error | context_window − 20K | User-facing error state |
| Auto-compact trigger | context_window − 13K buffer | Proactive compaction |
| Blocking limit | context_window − MAX_OUTPUT_TOKENS | Stops all sends |
| Summary headroom | 20K reserved | Output buffer for summary itself |

### Three Compaction Modes

1. **Microcompact** — Runs on every API call. Clears tool results/uses based on API schema (`clear_tool_uses_20250919`). Token-budgeted (50K tokens max for restored content).
2. **Session Memory Compaction** — Experimental LLM-free path. Summarizes into session memory file, keeps recent messages by token/text-block count thresholds. Falls back to legacy if session memory is empty or threshold-exceeding.
3. **Full Conversation Compaction** — Traditional LLM summarization. Streaming with retry, hook system, file/skill/plan/async-agent restoration, and PTL escape hatch.

### Feature Flags (Gradual Rollout)

| Flag | Effect |
|------|--------|
| `CONTEXT_COLLAPSE` | Disables auto-compact (context-collapse system takes over) |
| `REACTIVE_COMPACT` | Suppresses proactive compaction; relies on API 413 fallback |
| `PROMPT_CACHE_BREAK_DETECTION` | Resets prompt cache read baseline after compaction |
| `tengu_session_memory` | Enables experimental session memory compaction |
| `tengu_sm_compact` | Enables time-based microcompact config |
| `tengu_slate_heron` | Remote config for time-based gap threshold |

### Safety Mechanisms

- **Circuit breaker**: After 3 consecutive compaction failures, auto-compact is suspended for the session (~250K avoided API calls/day historically).
- **Recursion guards**: Prevents compact→subagent→compact deadlocks (`compact`, `session_memory`, and `marble_origami` agents bypass auto-compact).
- **PTL escape hatch** (`compact.ts`): If streaming hits `prompt_too_long`, aggressively truncates head messages and retries up to 3 times with exponential message pruning.
- **API invariant preservation** (`sessionMemoryCompact.ts`): Never splits `tool_use`/`tool_result` pairs across the compaction boundary, even when streaming yields separate messages with the same `message.id`.
