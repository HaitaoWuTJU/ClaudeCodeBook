# Summary of `services/PromptSuggestion/`

## Purpose of `PromptSuggestion/`

This directory implements a **proactive suggestion and speculative execution system** for Claude Code. When a user hasn't typed anything, the system generates a contextual prediction of what they might want to do next, executes that action speculatively in the background, and presents it for instant acceptance—or silently discards it if the user does something else instead.

## Contents Overview

| File | Primary Responsibility |
|------|------------------------|
| `promptSuggestion.ts` | Generates contextual suggestions using a forked agent with a specialized prompt, then filters them through 12 criteria to ensure quality and appropriateness. |
| `speculation.ts` | Executes suggested actions in an isolated background process using copy-on-write file overlays, enabling instant application if the user accepts or safe discard if they decline. |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                    User Idle (TUI State)                     │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  promptSuggestion.ts                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  generateSuggestion()                               │   │
│  │  ├─ Runs forked agent with SUGGESTION_PROMPT        │   │
│  │  └─ Instructs model to "stay silent" if unclear     │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  shouldFilterSuggestion()                            │   │
│  │  ├─ 2–12 word limit                                  │   │
│  │  ├─ Denies: meta-text, errors, questions, Claude     │   │
│  │  │        voice, single-words (except allowlist)     │   │
│  │  └─ Validates user input format consistency          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────────┐
│  Suggestion Rejected     │     │  Suggestion Accepted         │
│  (user types something   │     │  (speculation ready or      │
│   or suggestion filtered)│     │   already complete)         │
└─────────────────────────┘     └──────────────┬──────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  speculation.ts                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  startSpeculation()                                        │  │
│  │  ├─ Creates overlay dir: /tmp/claude-speculation/{uuid}   │  │
│  │  ├─ Spawns forked agent with canUseTool guard             │  │
│  │  ├─ Copy-on-write: edits redirect to overlay              │  │
│  │  └─ Tracks writtenPaths and message boundary              │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  handleSpeculationAccept()                                 │  │
│  │  ├─ copyOverlayToMain(): promote changes to cwd            │  │
│  │  ├─ setMessages(): inject speculated turns                 │  │
│  │  ├─ mergeFileStateCaches(): sync read-file cache          │  │
│  │  └─ Pipelining: generate next suggestion in background     │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Shared Dependencies
Both files co-depend on several core utilities:

| Module | Used By | Purpose |
|--------|---------|---------|
| `../../utils/forkedAgent.js` | Both | `runForkedAgent`, `createCacheSafeParams` for isolated execution |
| `../../state/AppStateStore.js` | Both | Speculation state, suggestion state, app-level state |
| `../analytics/index.js` | Both | `logEvent` for telemetry |
| `../../utils/hooks/postSamplingHooks.js` | Both | `REPLHookContext` for message context |

### Data Flow Across Files
1. **`promptSuggestion.ts`** produces a suggestion string and writes it to `AppState.suggestion`
2. **`speculation.ts`** reads that suggestion, starts a forked execution, and updates `AppState.activeSpeculation`
3. On acceptance, **`speculation.ts`** injects messages and logs analytics
4. **`promptSuggestion.ts`** logs suppression/acceptance metrics that include `timeSavedMs` from speculation

## Key Takeaways

### Architecture
- **Two-stage prediction**: Generation (promptSuggestion) → Execution (speculation) with clean separation of concerns
- **Forked isolation**: Both suggestion generation and speculation run in separate forked processes, never blocking the main loop
- **Copy-on-write safety**: File modifications during speculation never touch the real working directory until explicit acceptance

### Performance Considerations
- **Cache key preservation**: The forked agent deliberately avoids overriding `effortValue`, `maxOutputTokens`, or tool/thinking settings—empirically shown to cause 45× cache write spike (PR #18143)
- **Pipelining**: The next suggestion is generated while speculation runs, hiding latency
- **Rate limiting**: Suggestions respect external user rate limits and elicitation states

### User Experience
- **Instant feedback**: If speculation completes before user decides, actions appear instantly on acceptance
- **Silent discard**: Users never see rejected suggestions; all filtering happens server-side
- **ANT telemetry**: Time savings and acceptance rates are tracked for the ANT user base

### Security & Guardrails
- **Tool denylist**: Speculation restricts tool access based on permission mode
- **Read-only bash validation**: Bash commands must pass `checkReadOnlyConstraints`
- **12-point suggestion filter**: Prevents awkward, evaluative, or Claude-voiced suggestions

### Feature Flags
- Speculation gated by `USER_TYPE === 'ant'` and `SPECULATE_SUGGESTIONS=true`
- Prompt suggestions gated by GrowthBook features and non-interactive mode checks
