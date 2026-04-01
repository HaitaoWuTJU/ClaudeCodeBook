# Summary of `components/FeedbackSurvey/`

## Purpose of `FeedbackSurvey/`

This directory implements a **multi-trigger feedback collection system** for Claude Code's CLI. It captures user sentiment through surveys, optionally prompts for transcript sharing (restricted to internal "ant" environment), and submits data to Anthropic's analytics pipeline.

## Contents Overview

| File | Role |
|------|------|
| `index.tsx` | Public API — exports `useFeedbackSurvey` and `usePostCompactSurvey` hooks |
| `useFeedbackSurvey.tsx` | Trigger logic: shows survey when messages mention "memory"/"memories" (20% probability) |
| `usePostCompactSurvey.tsx` | Trigger logic: shows survey after session memory compaction (20% probability, deferred until next message) |
| `useSurveyState.tsx` | State machine: manages `'closed' → 'open' → 'thanks/transcript_prompt' → 'submitting' → 'submitted'` transitions |
| `FeedbackSurvey.tsx` | Renders survey UI based on current state (survey view, transcript prompt, thanks message) |
| `FeedbackSurveyView.tsx` | Shows rating options (Bad, Fine, Good, Dismiss) with digit input (0–3) |
| `TranscriptSharePrompt.tsx` | Asks user to share transcript (Yes, No, Don't ask again) with digit input (1–3) |
| `submitTranscriptShare.ts` | Collects messages + subagent transcripts, redacts sensitive info, POSTs to Anthropic API |
| `useDebouncedDigitInput.ts` | Shared hook: debounces single-digit keyboard input for survey responses |
| `utils.ts` | Types (`FeedbackSurveyResponse`, `TranscriptShareResponse`) and helpers |

## How Files Relate to Each Other

```
Consumer (e.g., root component)
    │
    ├── useFeedbackSurvey()        usePostCompactSurvey()
    │        │                              │
    │        ▼                              ▼
    │   useSurveyState()              useSurveyState()
    │        │                              │
    │        ▼                              ▼
    │   FeedbackSurvey                   FeedbackSurvey
    │        │                              │
    │        ├──────────────────────────────┤
    │        │                              │
    │        ▼                              ▼
    │   FeedbackSurveyView           FeedbackSurveyView
    │   (rating options 0-3)         (same component)
    │        │
    │        ▼
    │   useDebouncedDigitInput()
    │
    ├── TranscriptSharePrompt
    │        │
    │        ▼
    │   useDebouncedDigitInput()
    │
    └── submitTranscriptShare()
             │
             ▼
         Anthropic API (ant environment only)
```

**Trigger → State → UI → Submit pipeline:**

1. **Trigger hooks** (`useFeedbackSurvey`, `usePostCompactSurvey`) check Statsig gates, environment variables, policy limits, and probability thresholds — then call `open()` from `useSurveyState`
2. **State machine** (`useSurveyState`) manages transitions, stores `lastResponse`, and routes to either the transcript prompt or thanks flow
3. **UI components** (`FeedbackSurvey`, `FeedbackSurveyView`, `TranscriptSharePrompt`) render based on state and consume `useDebouncedDigitInput` for keyboard input
4. **Submission** (`submitTranscriptShare`) is invoked when user consents to transcript sharing

## Key Takeaways

- **Multi-trigger design**: Separate hooks handle different contexts (memory mentions vs. compaction) with shared `useSurveyState` and UI components — avoids duplicating state logic
- **Probabilistic sampling**: Both triggers use `Math.random() < 0.2` (20%) to limit survey frequency per session
- **Deferred timing**: Post-compact survey waits for the *next* message to appear after compaction before showing, avoiding mid-operation interruptions
- **Transcript sharing is ant-only**: `submitTranscriptShare` is gated by checking `"external" === 'ant'` in callers; the API endpoint is `api.anthropic.com`
- **Safety guards**: Transcript reads are size-gated (`MAX_TRANSCRIPT_READ_BYTES`), auth tokens are refreshed before API calls, and all errors are swallowed — never throws
- **Clean separation**: Trigger logic, state management, UI rendering, and data submission are decoupled, enabling testability and reuse
- **Input debouncing**: Single hook (`useDebouncedDigitInput`) handles digit validation and debouncing for both survey ratings and transcript prompt responses
