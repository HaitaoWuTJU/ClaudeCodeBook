# Summary of `tools/WebSearchTool/`

## Purpose of `WebSearchTool/`

This directory implements a complete web search tool that enables Claude to perform web searches by calling the Anthropic API. The tool is designed for CLI environments using the Ink library and supports streaming, progress updates, and domain filtering.

## Contents Overview

| File | Purpose |
|------|---------|
| `prompt.ts` | Generates the instruction prompt template that tells Claude how to format web search results, including mandatory "Sources:" section requirements |
| `UI.tsx` | Provides React UI rendering functions for CLI display: tool usage messages, progress updates ("Searching..."), and formatted results with duration |
| `WebSearchTool.ts` | Implements the core tool logic: input validation (Zod schemas), streaming to Anthropic API, response transformation, and provider support detection |

## How Files Relate to Each Other

```
prompt.ts
   │
   │  (provides instruction string)
   ▼
WebSearchTool.ts ─────────────────────────────────────────────────┐
   │                                                             │
   │  (transforms API results)                                   │
   ▼                                                             │
UI.tsx                                                           │
   │                                                             │
   │  (renders tool usage message, progress, results)            │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
              Final output displayed to user
```

**Data Flow**:

1. `WebSearchTool.ts` uses `prompt.ts` to construct the tool schema sent to the API
2. `WebSearchTool.ts` transforms API responses and streams progress events
3. `UI.tsx` consumes these events and renders:
   - Tool usage description (query + optional domain filters)
   - Progress messages during search
   - Final result with search count and duration
4. The tool enforces `prompt.ts` formatting requirements in its output block

## Key Takeaways

| Category | Details |
|----------|---------|
| **Capabilities** | Web search with max 8 searches per request; US-only; optional domain filtering |
| **Input Schema** | `query` (required), `allowed_domains`, `blocked_domains` |
| **Output Format** | Markdown links `[Title](URL)` with mandatory "Sources:" section |
| **Providers** | firstParty always, Vertex AI (Claude 4.0+), Foundry always |
| **Concurrency** | Safe for concurrent, read-only operations |
| **UI Framework** | Ink (React for CLIs) with Box/Text components |
| **Streaming** | Real-time progress via `query_update` and `search_results_received` events |
| **Time Display** | Automatically formats duration in seconds or milliseconds |
