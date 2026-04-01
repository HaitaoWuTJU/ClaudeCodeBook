# Summary of `services/toolUseSummary/`

## Purpose of `toolUseSummary/`

This directory provides a lightweight AI-powered service for generating human-readable summaries of completed tool executions. These summaries serve as high-level progress indicators in the SDK, giving clients a quick understanding of what actions the AI took during a session without needing to inspect individual tool calls.

## Contents Overview

The directory contains a single file:

| File | Description |
|------|-------------|
| `toolUseSummaryGenerator.ts` | Generates one-line summaries (≤30 characters) of tool batches using the Haiku AI model. Includes a system prompt template defining the output format, a truncation helper for tool inputs/outputs, and the main async `generateToolUseSummary()` function. |

## How Files Relate to Each Other

Since this directory contains only one file, there are no internal relationships between files. The `toolUseSummaryGenerator.ts` file is self-contained and exports a single public function `generateToolUseSummary()`. It depends on several internal utilities (`errors.js`, `log.js`, `slowOperations.js`, `systemPromptType.js`) and the `queryHaiku` API from the claude service.

## Key Takeaways

1. **Non-critical operation**: Failures are silently logged and return `null` — this feature never blocks or breaks the SDK if summarization fails.

2. **Output format**: Summaries follow a "git commit subject" style — past tense verb, distinctive noun, minimal articles:
   - ✅ `"Fixed NPE in UserService"`
   - ✅ `"Created signup endpoint"`
   - ❌ `"The AI assistant created a new signup endpoint for the application"`

3. **Efficiency considerations**:
   - Input truncation (300 chars per tool, 200 chars context)
   - Prompt caching enabled for repeated calls
   - Abort signal support for cancellation

4. **Single responsibility**: This module does one thing well — converts a batch of tool execution data into a concise human-readable label using a small, fast AI model (Haiku).
