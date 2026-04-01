# Summary of `tools/SleepTool/`

## Summary

This directory contains configuration and prompt constants for a **Sleep tool** used in an AI agent system.

### Purpose

The `SleepTool/` module provides utility for pausing agent execution for a specified duration. It enables the AI to wait idly when needed (e.g., when instructed to "sleep" or "rest"), while still being responsive to periodic check-ins via `<TICK_TAG>` prompts.

### Contents Overview

| File | Role |
|------|------|
| `index.ts` | Exports three string constants: `SLEEP_TOOL_NAME`, `DESCRIPTION`, and `SLEEP_TOOL_PROMPT` |

### How It Fits Into the System

- **Input**: Reads `TICK_TAG` constant from `../../constants/xml.js` to embed check-in tag reference in the prompt
- **Output**: Exports strings consumed by a tool registration or prompt injection system elsewhere in the codebase
- **Design notes**:
  - Prefers `SleepTool` over `Bash(sleep ...)` to avoid holding shell processes
  - Supports concurrent execution with other tools
  - Each wake-up costs an API call; prompt cache expires after 5 minutes of inactivity

### Key Takeaways

1. **Pure configuration** — no runtime logic, only exports static strings
2. **Self-contained prompt** — the `SLEEP_TOOL_PROMPT` includes all instructions the AI needs to use the tool correctly
3. **Tagged check-ins** — dynamically injects `TICK_TAG` so the AI knows how to recognize periodic prompts during sleep
4. **Cost-aware** — explicit guidance about API call costs and cache expiration embedded directly in the prompt
