# Summary of `services/SessionMemory/`

## Purpose of `services/SessionMemory/`

The `SessionMemory/` service automatically maintains a persistent markdown file with notes about the current Claude Code conversation. It runs as a post-sampling hook in the background using a forked subagent, extracting key information (task progress, files modified, errors encountered, learnings, etc.) without interrupting the main conversation flow. The extracted notes are stored in `~/.claude/session-memory/session.md` and can be referenced by the LLM in future turns for continuity across long sessions.

## Contents Overview

| File | Role | Key Responsibility |
|------|------|--------------------|
| `sessionMemory.ts` | **Feature entry point** | Registers the post-sampling hook, orchestrates extraction via forked agent, manages tool permission restrictions |
| `sessionMemoryUtils.ts` | **State & configuration utilities** | Avoids circular dependencies by exposing state getters/setters, threshold checks, config management, and extraction waiting logic |
| `prompts.ts` | **Templates & content processing** | Defines the 10-section memory template, builds update prompts with variable substitution, analyzes section token sizes, and handles truncation for compact messages |

## How Files Relate to Each Other

```
sessionMemory.ts
    │
    ├── imports sessionMemoryUtils.ts
    │       ├── getSessionMemoryConfig() ──── thresholds (init/update/tool-calls)
    │       ├── getLastSummarizedMessageId() ── tracking last extracted message
    │       ├── recordExtractionTokenCount() ── token snapshot for growth calc
    │       ├── isSessionMemoryInitialized() ── init flag
    │       ├── waitForSessionMemoryExtraction() ── poll for completion
    │       └── resetSessionMemoryState() ── test cleanup
    │
    ├── imports prompts.ts
    │       ├── DEFAULT_SESSION_MEMORY_TEMPLATE ── 10-section markdown skeleton
    │       ├── getDefaultUpdatePrompt() ── instruction text for the subagent
    │       ├── loadSessionMemoryTemplate() ── custom template from ~/.claude/session-memory/config/
    │       ├── loadSessionMemoryPrompt() ── custom prompt from ~/.claude/session-memory/config/
    │       ├── buildSessionMemoryUpdatePrompt() ── assembles full subagent prompt
    │       ├── analyzeSectionSizes() ── token counts per section for warnings
    │       ├── generateSectionReminders() ── oversized section warnings
    │       ├── substituteVariables() ── single-pass {{var}} replacement
    │       ├── isSessionMemoryEmpty() ── detect template-only content
    │       └── truncateSessionMemoryForCompact() ── cut sections for compact mode
    │
    └── calls runForkedAgent() with:
            • Isolated subagent context (no parent state mutation)
            • Tool restriction: only FileEditTool on the memory file
            • Prompt built from prompts.ts with context variables substituted
```

### Circular Dependency Resolution

`sessionMemoryUtils.ts` exists precisely to break a circular import chain:

```
sessionMemory.ts
    │
    ├── needs runAgent
    │
    └── prompts.ts
            │
            └── config dir lookup
                    │
                    └── (cycle back to sessionMemory.ts)

Solution: sessionMemoryUtils.ts exposes only pure functions and state
getters/setters, free of runAgent, so prompts.ts can safely import it.
```

## Key Takeaways

1. **Non-blocking, background extraction**: The feature gate and config use cached values that return immediately without blocking GrowthBook initialization. Values may be stale but are refreshed in the background.

2. **Safety-first triggering**: Extraction only fires when the last assistant turn has **no tool calls in flight**, preventing the subagent from seeing partially-completed operations. Tool call thresholds, token growth thresholds, and initialization thresholds must all align.

3. **Single-pass variable substitution** in prompts.ts prevents two bugs: `$` backreference corruption and double-substitution when user content contains `{{varName}}`.

4. **Template has 10 fixed sections**: Session Title, Current State, Task Specification, Files and Functions, Workflow, Errors & Corrections, Codebase and System Documentation, Learnings, Key Results, Worklog. The subagent is instructed to preserve headers and italic descriptions unchanged; only content below descriptions is editable.

5. **Hard token limits enforced**: `MAX_SECTION_LENGTH = 2000` tokens per section and `MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000` tokens total. When exceeded, prompts.ts generates truncation markers like `[... section truncated for length ...]`.

6. **Sandboxed execution**: The forked subagent runs in full isolation — no parent state mutation, no parent filesystem cache access, and all tools blocked except `FileEditTool` on the exact memory file path.

7. **Customizable via filesystem**: Users can drop `~/.claude/session-memory/config/template.md` and `~/.claude/session-memory/config/prompt.md` to override defaults without code changes.

8. **Compact mode support**: `truncateSessionMemoryForCompact()` handles ultra-long contexts by preserving headers, truncating at line boundaries, and returning a `wasTruncated` flag for callers to adjust behavior.
