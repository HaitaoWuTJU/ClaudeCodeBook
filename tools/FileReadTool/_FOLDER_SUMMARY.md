# Summary of `tools/FileReadTool/`

## Purpose of `FileReadTool/`

Implements the `Read` tool — a core Claude Code tool that enables an AI agent to read local files from the filesystem. It handles multiple file types (plain text, images, PDFs, Jupyter notebooks, agent output files) with support for pagination, token-based size limits, permission checking, and analytics tracking.

## Contents Overview

| File | Responsibility |
|------|---------------|
| **`FileReadTool.ts`** | Core implementation — input/output schemas, `callInner()` routing by file type, token validation, permission checks, analytics, and UI prompt rendering |
| **`imageProcessor.ts`** | Abstraction layer over image processing — dynamically loads a native `image-processor-napi` module in bundled mode, falls back to `sharp` otherwise; exposes resize, format conversion, and image creation |
| **`limits.ts`** | Centralized configuration for read limits (`maxTokens`, `maxSizeBytes`) — respects env var overrides, GrowthBook feature flags, and hardcoded defaults; memoized to prevent mid-session cap drift |
| **`prompt.ts`** | Static prompt templates and display constants — `DESCRIPTION`, `MAX_LINES_TO_READ`, line-format instruction, offset guidance, and `renderPromptTemplate()` which assembles the complete tool prompt with conditional PDF support |
| **`UI.tsx`** | React/Ink UI rendering for tool use messages, result summaries, and error messages — handles 6 distinct output types and special-cases agent output files with task ID tags |

## How Files Relate to Each Other

```
FileReadTool.ts (orchestrator)
    │
    ├── reads limits.ts
    │       └─ getDefaultFileReadingLimits() → maxTokens, maxSizeBytes
    │
    ├── reads prompt.ts
    │       ├─ renderPromptTemplate() uses isPDFSupported() and BASH_TOOL_NAME
    │       └─ exports: MAX_LINES_TO_READ, OFFSET_INSTRUCTION_*, LINE_FORMAT_INSTRUCTION
    │
    ├── calls imageProcessor.ts
    │       └─ getImageProcessor() / getImageCreator() for image resize/compress
    │
    └── calls UI.tsx
            └─ renderToolUseMessage(), renderToolResultMessage(),
               renderToolUseErrorMessage(), userFacingName(), getToolUseSummary()
```

**Flow**: `FileReadTool.ts` is the orchestrator — it reads configuration from `limits.ts`, uses `prompt.ts` to assemble the tool prompt and collect display strings, delegates image processing to `imageProcessor.ts`, and passes display data to `UI.tsx` for terminal rendering.

## Key Takeaways

1. **Multi-format support via routing**: `callInner()` dispatches based on file extension — `.ipynb` → notebook, image extensions → image processor, `.pdf` → PDF utilities, everything else → raw text with offset/limit pagination.

2. **Single-read image optimization**: `imageProcessor.ts` reads an image once into a buffer, then applies progressive compression from that same buffer rather than re-reading, avoiding redundant I/O.

3. **Token budget over byte budget**: Text files are capped by `maxSizeBytes` (total file size), but images are capped by `maxTokens` (output token count) and bypass the byte cap entirely. Token counting uses rough estimation first, with a live API call only when the estimate exceeds 25% of the limit.

4. **Memoized limits for session stability**: `getDefaultFileReadingLimits()` is memoized so that the GrowthBook feature flag is evaluated only once. Without this, a flag change mid-session could silently alter read behavior.

5. **Cyber risk mitigation**: For non-exempt models, a `<system-reminder>` is injected into text read responses, asking the model not to assist with improving malware. This is omitted from UI chrome to keep the terminal output clean.

6. **Agent output special-casing**: Files matching `{taskOutputDir}/tasks/{taskId}.output` are treated as agent outputs — the UI renders a task ID tag instead of a file path, and analytics are tagged as session file reads.

7. **macOS screenshot path resolution**: `FileReadTool.ts` normalizes thin spaces (U+202F) in macOS screenshot filenames before AM/PM, ensuring consistent path resolution across locales.

8. **Fallback image processing**: If the native `image-processor-napi` module is unavailable (e.g., in development or certain deployment environments), `imageProcessor.ts` silently falls back to `sharp`, with a console warning in non-production mode.
