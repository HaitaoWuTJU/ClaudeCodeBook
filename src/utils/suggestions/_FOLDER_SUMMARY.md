# Summary of `utils/suggestions/`

## Purpose of `suggestions/`

Provides a unified, extensible autocomplete system for slash commands and file paths within a prompt input interface. Each module addresses a distinct suggestion domain—slash commands, directory/file paths, shell history, skill usage rankings, and Slack channels—while sharing consistent patterns for caching, result formatting, and integration with the `PromptInputFooterSuggestions` UI component.

## Contents Overview

| File | Primary Responsibility |
|------|------------------------|
| `commandSuggestions.ts` | Fuzzy search for slash commands (`/commit`, `/voice-memo`) with Fuse.js; handles mid-input detection, usage-based ranking, and command application |
| `directoryCompletion.ts` | Filesystem path completions; scans directories with LRU caching, returns both file and directory suggestions |
| `shellHistoryCompletion.ts` | Shell history autocomplete for `!`-prefixed commands; caches history for 60s |
| `skillUsageTracking.ts` | Tracks skill invocation frequency/recency; provides exponential-decay scoring to rank frequently-used skills higher |
| `slackChannelSuggestions.ts` | Slack MCP server integration; queries `slack_search_channels`, caches results with prefix reuse, tracks confirmed-real channel names |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────┐
│                    Prompt Input (UI Layer)                      │
│          PromptInputFooterSuggestions ← SuggestionItem[]        │
└──────────────────────────────────────────────────────────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
┌─────────────────┐  ┌─────────────────────┐  ┌─────────────────┐
│ commandSugges-  │  │ directoryCompletion │  │ slackChannel-   │
│ tions.ts        │  │ / pathCompletion    │  │ Suggestions.ts  │
│                 │  │                     │  │                 │
│ ┌─────────────┐ │  │ ┌─────────────────┐ │  │ ┌─────────────┐ │
│ │ Fuse.js     │ │  │ │ LRU cache (500) │ │  │ │ MCP server │ │
│ │ fuzzy search│ │  │ │ 5-min TTL      │ │  │ │ connection │ │
│ └─────────────┘ │  │ └─────────────────┘ │  │ └─────────────┘ │
│ ┌─────────────┐ │  │                     │  │                 │
│ │ skillUsage- │ │  │                     │  │                 │
│ │ Tracking.ts │ │  │                     │  │                 │
│ │ (shared)    │ │  │                     │  │                 │
│ └─────────────┘ │  │                     │  │                 │
└─────────────────┘  └─────────────────────┘  └─────────────────┘
          │
          ▼
┌─────────────────────────────────────────┐
│     shellHistoryCompletion.ts           │
│     (triggered by '!' prefix)           │
└─────────────────────────────────────────┘
```

### Data Flow (End-to-End Example)

A user typing `/cl` in the prompt input triggers:

1. **`commandSuggestions.ts` → `isCommandInput()`** validates the `//` prefix
2. **`hasCommandArgs()`** checks for trailing arguments
3. **`getCommandFuse()`** returns the cached Fuse.js index
4. **Fuse.js fuzzy search** ranks commands: `claude-` commands score highest due to `commandName` weight of 3
5. **`generateCommandSuggestions()`** categorizes and slices results
6. **`SuggestionItem[]`** returned to `PromptInputFooterSuggestions`
7. On selection: **`applyCommandSuggestion()`** updates input, optionally executing if no args required

## Key Takeaways

| Pattern | Implementation |
|---------|----------------|
| **Caching** | Each suggestion domain maintains its own cache with domain-appropriate TTLs and eviction strategies (Fuse identity-based, LRU 500 items / 5 min, debounced Map, 50-entry Map) |
| **Fuzzy Search** | Fuse.js used only for commands; paths use simple prefix/filtering |
| **Result Limits** | Commands: none (categorized); Paths: 100 cap + 10 default; Slack: 20 MCP + 10 display |
| **User Input Parsing** | Mid-input slash command detection with whitespace awareness; path recognition for `~/`, `/`, `./`, `../`, `~`, `.`, `..` |
| **Debouncing** | Skill usage: 60s; Slack MCP: shared `inflightQuery` guard |
| **Reactive Signaling** | Slack module emits `knownChannelsChanged` signal; Fuse index uses command array identity |
| **Ranking Factors** | Commands: Fuse score + usage score; Paths: directories first, then alphabetical; Slack: MCP order |
| **Edge Cases** | Hidden commands (exact name prepended if no visible duplicate), shell history (filters `!`-prefixed only), path cache reuse (prefix matching) |
