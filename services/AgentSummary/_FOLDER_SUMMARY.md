# Summary of `services/AgentSummary/`

## AgentSummary/ Directory

### Purpose

The `AgentSummary/` directory provides a **periodic background summarization service** for coordinator mode sub-agents. It enables real-time progress tracking by forking the sub-agent's conversation on a timer (~30 seconds) to generate concise progress summaries for UI display.

### Contents Overview

| File | Purpose |
|------|---------|
| `agentSummary.ts` | Core implementation containing timer-based summarization logic, prompt builder, and summary update mechanism |

### How Components Relate

```
AgentTool.tsx (parent)
        │
        │ passes cacheSafeParams, agentId, taskId
        ▼
startAgentSummarization()
        │
        ├──► setInterval (30s)
        │         │
        │         ▼
        │    runSummary()
        │         │
        │         ├──► getAgentTranscript() ──► Transcript
        │         │
        │         ├──► filterIncompleteToolCalls() ──► cleanMessages
        │         │
        │         └──► runForkedAgent() ──► updateAgentSummary()
        │
        └──► returns stop() control
```

The service integrates with:
- **`LocalAgentTask.js`** — receives summary updates via `updateAgentSummary()`
- **`sessionStorage.js`** — reads agent transcripts via `getAgentTranscript()`
- **`runAgent.js`** — filters incomplete tool calls for clean context
- **`forkedAgent.js`** — spawns summary-generation forks

### Key Takeaways

1. **Decoupled from main execution**: Summarization runs asynchronously in a timer loop, not blocking the main agent loop.

2. **Prompt cache optimization**: Uses `canUseTool` callback to deny tools without breaking cache sharing with the parent agent.

3. **Stale-message prevention**: Drops `forkContextMessages` from closure; re-reads fresh transcript each tick.

4. **UI-ready summaries**: Output is constrained to 3-5 words in present tense, suitable for progress indicators.

5. **Clean abort handling**: `stop()` cancels both the timer and any in-flight summary via `signal.abort`.
