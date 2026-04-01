# Summary of `tools/GrepTool/`

## Purpose of `GrepTool/`

Implements a ripgrep-powered search tool for AI agents that enables file content searching using regular expressions. The tool provides a secure, configurable interface with multiple output modes, permission-based access control, and formatted terminal display.

## Contents Overview

| File | Type | Description |
|------|------|-------------|
| `GrepTool.ts` | Main Implementation | Core tool logic: input validation, ripgrep execution, output processing, permission checks |
| `prompt.ts` | Prompt Definition | Usage instructions and capability description for AI agents |
| `UI.tsx` | React/UI | Ink-based terminal components for rendering search results in different modes |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                     GrepTool Module                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  prompt.ts ─────────────┐                                   │
│  (GREP_TOOL_NAME,       │                                   │
│   getDescription)       │                                   │
│                    ▼    │    GrepTool.ts                    │
│                 ┌───────────────────────┐                  │
│                 │  buildTool()          │                  │
│                 │  - inputSchema        │                  │
│                 │  - outputSchema       │                  │
│                 │  - validateInput()     │                  │
│                 │  - checkPermissions()  │                  │
│                 │  - call()             │──────────────────┼──► ripgrep process
│                 │  - mapToolResult...() │                  │
│                 └───────────────────────┘                  │
│                              │                              │
│                              ▼                              │
│  UI.tsx ─────────────┐  Output                            │
│  (renderTool*(),     │──────────────┐                      │
│   SearchResultSummary, │           │                      │
│   getToolUseSummary)  │           ▼                      │
│                      └─────────────────────────────┐        │
│                      React/Ink Terminal Display   │        │
│                      (Box, Text, CtrlOToExpand)   │        │
│                      └─────────────────────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Key Takeaways

| Category | Details |
|----------|---------|
| **Purpose** | Wraps ripgrep for AI agent file content search with regex support |
| **Output Modes** | `content` (lines), `files_with_matches` (paths), `count` (totals) |
| **Security** | UNC path protection, VCS directory exclusion, permission-based ignore patterns |
| **Defaults** | 250 result limit, hidden files included, max 500 column width |
| **Display** | Ink-based terminal UI with verbose/compact modes, Ctrl+O expand, pluralization |
| **Integration** | Uses `buildTool()` framework, respects file permissions, auto-classification support |
