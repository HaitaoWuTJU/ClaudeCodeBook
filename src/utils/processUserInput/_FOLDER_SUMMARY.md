# Summary of `utils/processUserInput/`

## Purpose of `processUserInput/`

This directory is the **entry point for all user-generated input** in the application. Every piece of content the user types or pastes — whether a plain text message, a slash command like `/commit`, a bash command prefixed with `!`, or an image — flows through this module. The directory orchestrates parsing, validation, image processing, hook execution, and routing to the appropriate handler, ultimately producing a normalized set of messages ready for submission to the AI model.

---

## Contents Overview

| File | Role | Output |
|------|------|--------|
| **`processUserInput.ts`** | Central orchestrator. Receives raw user input, normalizes images, extracts attachments, detects ultraplan keywords, runs `UserPromptSubmit` hooks, and routes to the correct handler. | `ProcessUserInputBaseResult` |
| **`processTextPrompt.ts`** | Handles plain text prompts (CLI string or SDK `ContentBlockParam[]`). Creates `UserMessage` objects, detects negative/keep-going keywords, and emits OTel telemetry. | `{ messages[], shouldQuery: true }` |
| **`processSlashCommand.tsx`** | Handles slash commands (e.g., `/commit`, `/task`). Parses command + args, validates against the registry, and executes synchronously or in a forked background sub-agent via `runAgent()`. | `{ messages[], shouldQuery, command, resultText }` |
| **`processBashCommand.tsx`** | Handles bash or PowerShell commands. Routes to `BashTool` or `PowerShellTool`, streams real-time progress to the UI, formats stdout/stderr into XML-wrapped messages. | `{ messages[], shouldQuery: false }` |

---

## How Files Relate to Each Other

```
User Input (string | ContentBlockParam[])
         │
         ▼
┌─────────────────────────────────┐
│ processUserInput.ts             │  ◄─── Entry point
│  - Normalize/resize images      │
│  - Extract attachments          │
│  - Detect ultraplan keywords     │
│  - Execute UserPromptSubmit hooks│
└────────────┬────────────────────┘
             │ Routes by type
     ┌───────┼───────┬────────────┐
     ▼       ▼       ▼
┌─────────┐ ┌──────┐ ┌────────────┐
│ starts  │ │ mode │ │  default   │
│  with   │ │ ===  │ │  (plain    │
│   '/'   │ │bash  │ │  text)     │
└────┬────┘ └──┬───┘ └─────┬──────┘
     │         │           │
     ▼         ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────────┐
│ process  │ │ process  │ │ process     │
│ Slash    │ │ Bash     │ │ TextPrompt  │
│ Command  │ │ Command  │ │             │
└────┬─────┘ └────┬────┘ └──────┬──────┘
     │            │             │
     ▼            ▼             ▼
  messages    messages      messages
  + resultText + stdout    + shouldQuery
  + shouldQuery / stderr     (true)
```

**Dependency chain (imports):**

- `processUserInput.ts` imports and calls all three handler files
- `processBashCommand.tsx` imports `processToolResultBlock` from sibling `utils/toolResultStorage.js`
- `processSlashCommand.tsx` imports command registry from `src/commands.js` and sub-agent runner from `src/tools/AgentTool/runAgent.js`
- `processTextPrompt.ts` imports message creators from `utils/messages.js` and OTel utilities from `utils/telemetry/`

---

## Key Takeaways

1. **Three distinct input modes.** The system partitions user input into three processing paths: slash commands (prefixed `/`), bash/shell commands (prefixed `!` or `mode: bash`), and everything else. Each path has its own message output contract — slash commands may return a `resultText` and `command` field; bash commands always return `shouldQuery: false`; plain prompts always return `shouldQuery: true`.

2. **Images are processed before routing.** All image blocks are normalized, resized, and stored to disk *before* the routing decision, so that resized image references propagate into whichever handler is chosen.

3. **Hooks run *after* base processing, *before* submission.** The `UserPromptSubmit` hook chain executes in `processUserInput.ts` after the base handler returns but before the final `ProcessUserInputBaseResult` is emitted. Hooks can block, augment, or replace the prompt.

4. **Background command execution via Kairos.** When the Kairos feature flag is enabled, slash commands launch as fire-and-forget forked sub-agents whose results re-enter the message queue as hidden meta-prompts, enabling parallel startup command execution without serial subagent chains.

5. **Shell routing is gated by platform and env.** PowerShell is only used when `isPowerShellToolEnabled()` returns true and the resolved shell is `powershell`; this also triggers lazy-loading of the ~300 KB `PowerShellTool` module.
