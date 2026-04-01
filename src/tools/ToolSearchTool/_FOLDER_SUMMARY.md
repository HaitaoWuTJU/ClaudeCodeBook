# Summary of `tools/ToolSearchTool/`

## ToolSearchTool/ Directory Summary

### Purpose

The `ToolSearchTool/` directory implements a **deferred tool discovery system** for the Kairos CLI. Rather than pre-loading all available tool JSONSchemas into the prompt (which would exceed context limits), this system:

1. **Announces** which tools are available but deferred
2. **Allows the AI model** to request tool schemas on-demand via a `select:` directive or keyword search
3. **Searches and ranks** tools by relevance when the model provides a query

In essence, it transforms a static list of ~100+ tools into an interactive search interface embedded in the system prompt.

---

### Contents Overview

| File | Purpose |
|------|---------|
| `constants.ts` | Exports the single named constant `TOOL_SEARCH_TOOL_NAME = 'ToolSearch'` used as the tool's canonical identifier. |
| `prompt.ts` | Defines the system prompt text and core deferral logic. Contains `isDeferredTool()` (which tools should be lazy-loaded), `getToolLocationHint()` (where deferred tools appear in output), `formatDeferredToolLine()` (rendering), and `getPrompt()` (assembly). Also describes the `select:` and keyword query interfaces to the model. |
| `ToolSearchTool.ts` | The main tool implementation. Handles `select:` direct selection and keyword-based search with relevance scoring, manages a memoized description cache, tracks pending MCP servers, and logs search outcome analytics. |

---

### How Files Relate to Each Other

```
constants.ts ──────────────────────────────► TOOL_SEARCH_TOOL_NAME
      │                                              │
      │ (exports name for use in)                    │ (identifies this tool in
      │                                              │  isDeferredTool check)
      ▼                                              ▼
prompt.ts ◄───────────────────────────────── ToolSearchTool.ts
      │                                                      │
      │ (provides getPrompt() for rendering)                │ (calls getPrompt() and
      │                                                      │  isDeferredTool())
      │                                                      │
      │         ToolSearchTool.ts ◄──────────────────────────┘
      │                   │
      │                   │ (provides TOOL_SEARCH_TOOL_NAME for
      │                   │  isDeferredTool skip-check)
      ▼                   ▼
┌─────────────────────────────────────┐
│     Main Tool Entry Point           │
│  (buildTool)                        │
└─────────────────────────────────────┘
```

**Data flow of a typical search:**
1. `ToolSearchTool` is built using `buildTool()` with schemas from Zod and prompt from `prompt.ts`.
2. At init, `prompt.ts` checks which tools are deferred (excluding Agent, Brief, SendUserFile, and ToolSearch itself).
3. The model, given the prompt, may invoke `ToolSearchTool` with either `select:ToolName` (direct) or a keyword query.
4. `ToolSearchTool.call()` searches/filters the deferred tools and returns ranked results with analytics.

---

### Key Takeaways

| Takeaway | Detail |
|----------|--------|
| **Deferred by default** | Tools are assumed deferred unless `alwaysLoad`, `isMcp=false` + `shouldDefer=false`, or explicitly exempted. |
| **Feature-gated exemptions** | `KAIROS`, `KAIROS_BRIEF`, and `FORK_SUBAGENT` flags control whether Agent, Brief, SendUserFile, and ForkSubagent tools are deferred or eagerly loaded. |
| **Two search modes** | `select:name1,name2` for direct lookup (O(n) name match) and free-text keyword query (O(m·n) scoring across names, hints, descriptions). |
| **Scoring is weighted** | Name exact match (10–12 pts) >> partial name (5–6 pts) >> searchHint (4 pts) >> description (2 pts). |
| **MCP tools are always deferred** | MCP tools have `isMcp: true`, bypassing the `shouldDefer` flag check entirely. Pending servers are surfaced when searches return zero results. |
| **Dead code elimination** | `BRIEF_TOOL_NAME` and `SEND_USER_FILE_TOOL_NAME` are lazy-required only when their feature flags are active, enabling tree-shaking. |
| **Search hints disabled** | An A/B experiment (`exp_xenhnnmn0smrx4`) showed no benefit from `tool.searchHint`, so `formatDeferredToolLine` now only outputs the tool name. |
| **Platform-specific output** | Results are formatted as `tool_reference` blocks for 1P/Foundry; plain text for Bedrock/Vertex. |
