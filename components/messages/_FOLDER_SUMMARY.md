# Summary of `components/messages/`

## Purpose of `messages/`

This directory is the **message rendering layer** for a CLI/terminal interface built with [Ink.js](https://github.com/vadimdemedes/ink) (React for CLIs). It handles every class of message that can appear in a Claude Code conversation turn — advisor calls, assistant thinking, text responses (with error handling), tool execution results (success/error/cancel/reject), teammate messages, resource updates, teammate refusals, and user prompts. The directory is structured around **message-type routing**: a top-level component inspects incoming block data and delegates rendering to the smallest specialized component possible.

---

## Contents Overview

| File / Directory | Role |
|---|---|
| **`index.tsx`** | Barrel re-export for the directory |
| **`AdvisorMessage.tsx`** | Renders advisor tool invocations and results (`server_tool_use`, `advisor_result`, `advisor_tool_result_error`, `advisor_redacted_result`) |
| **`AssistantRedactedThinkingMessage.tsx`** | Renders the dim/italic "✻ Thinking…" indicator for redacted assistant thinking |
| **`AssistantTextMessage.tsx`** | Renders assistant text responses, handling 15+ distinct error cases (rate limits, API key failures, org disables, timeouts, billing, etc.) and falling back to `<Markdown>` for normal text |
| **`AssistantToolErrorMessage.tsx`** | Renders assistant-originated tool errors; routes by error prefix to `InterruptedByUser`, `RejectedPlanMessage`, `RejectedToolUseMessage`, or a custom tool renderer |
| **`AssistantToolResultMessage.tsx`** | Renders assistant-originated tool successes (e.g., web search results); validates output schema, renders via `tool.renderToolResultMessage()`, shows tool use details (name, args, ID), and manages classifier approval badge loading |
| **`AssistantBashCommandMessage.tsx`** | Parses XML-tagged bash output (`<bash-stdout>`, `<bash-stderr>`) from assistant text blocks and renders with appropriate color coding (green for stdout, red for stderr, bold dim for header) |
| **`AssistantResourceUpdateMessage.tsx`** | Renders MCP resource polling/refresh events with a `↻` refresh arrow in the header |
| **`AssistantTeammateMessage.tsx`** | Parses XML-tagged teammate messages; routes to `AssistantPlanApprovalMessage`, `AssistantShutdownMessage`, `AssistantTaskAssignmentMessage`, `AssistantTaskCompletedMessage`, or plain content with ANSI rendering |
| **`TeammateMessage.tsx`** (nested) | Renders leader-originated teammate messages; parses `<teammate-message>` XML tags, filters shutdown/termination/idle notifications, routes to specialized sub-components |
| **`UserToolResultMessage/`** | Complete tool-result subsystem (see its own section below) |
| **`UserTeammateMessage.tsx`** | Renders user-side teammate messages; parses `<teammate-message>` XML tags, filters shutdown/termination/idle notifications, routes to plan approval, shutdown, task assignment, or `TeammateMessageContent` |
| **`UserTextMessage.tsx`** | Top-level router for user text blocks; routes to 15+ possible sub-components based on content prefixes (`<bash-stdout>`, `<COMMAND_MESSAGE_TAG>`, `<teammate-message>`, `<mcp-resource-update>`, etc.), falling back to `UserPromptMessage` |
| **`UserPromptMessage.tsx`** | Renders a plain user text prompt with optional top margin, using `MessageResponse` as a wrapper |

---

## How Files Relate to Each Other

```
UserTextMessage (top-level user dispatcher)
│
├─ UserTeammateMessage ──► PlanApprovalMessage / ShutdownMessage / TaskAssignmentMessage / TaskCompletedMessage / TeammateMessageContent
│
├─ UserToolResultMessage/ (user-initiated tool results)
│   │
│   ├── UserToolSuccessMessage ──► tool.renderToolResultMessage() + HookProgressMessage + SentryErrorBoundary
│   ├── UserToolErrorMessage ──► InterruptedByUser / RejectedPlanMessage / RejectedToolUseMessage / FallbackToolUseErrorMessage
│   ├── UserToolCanceledMessage ──► InterruptedByUser
│   ├── UserToolRejectMessage ──► FallbackToolUseRejectedMessage / tool.renderToolUseRejectedMessage()
│   └── utils.tsx ──► useGetToolFromMessages hook
│
└─ UserPromptMessage (generic fallback)

AssistantTextMessage (top-level assistant dispatcher)
│
├─ AssistantToolErrorMessage ──► InterruptedByUser / RejectedPlanMessage / RejectedToolUseMessage / FallbackToolUseErrorMessage
│
├─ AssistantToolResultMessage ──► tool.renderToolResultMessage() + ToolUseDetails + ClassifierApprovalMessage (lazy)
│
├─ AssistantBashCommandMessage ──► (XML-tagged bash output from text blocks)
│
├─ AssistantResourceUpdateMessage ──► (XML-tagged resource polling events)
│
├─ AssistantTeammateMessage ──► AssistantPlanApprovalMessage / AssistantShutdownMessage / AssistantTaskAssignmentMessage / AssistantTaskCompletedMessage
│
└─ AssistantRedactedThinkingMessage ──► (✻ Thinking… indicator)

AdvisorMessage (separate advisor tree)
│
└─ ToolUseLoader / CtrlOToExpand / MessageResponse
```

**Flow from a message block to pixels:**

1. A `NormalizedUserMessage` or `NormalizedAssistantMessage` arrives with typed `block`s (`TextBlockParam`, `ToolResultBlockParam`, etc.).
2. `UserTextMessage` or `AssistantTextMessage` receives the message, inspects the `param.text` string or `param.content` object.
3. Content-based routing (string `startsWith`, `includes`, `===`) dispatches to the smallest relevant component.
4. Tool-result components resolve a `Tool` definition via `useGetToolFromMessages` (from the message store) and call its render methods (`renderToolResultMessage`, `renderToolUseErrorMessage`, etc.).
5. Error types are handled with prefix-based routing — special sentinel strings (`CANCEL_MESSAGE`, `INTERRUPT_MESSAGE_FOR_TOOL_USE`, `PLAN_REJECTION_PREFIX`, etc.) route to `InterruptedByUser`, `RejectedPlanMessage`, `RejectedToolUseMessage`, etc.
6. Everything renders through Ink (`<Box>`, `<Text>`, `<Ansi>`) and `MessageResponse` (the consistent wrapper), ultimately producing a styled terminal output.

---

## Key Takeaways

### Routing Architecture
- **Two-level dispatch** (`UserTextMessage` → `UserToolResultMessage` → sub-components, `AssistantTextMessage` → `AssistantToolErrorMessage`/`AssistantToolResultMessage`) keeps each file small and focused.
- **String prefix matching** (not type discrimination) is the primary routing mechanism for user text: the same `TextBlockParam.text` field carries XML tags (`<bash-stdout>`, `<teammate-message>`, `<COMMAND_MESSAGE_TAG>`) that act as routing signals.
- **Boolean + prefix dispatch** for tool results: `param.is_error` or `param.content` prefixes distinguish success, error, cancel, and reject states.

### Error Handling Depth
- `AssistantTextMessage` handles ~15 distinct named error cases (API key, rate limit, org disable, timeout, billing, token revoked, prompt too long, user abort, etc.) with specific copy and suggested actions (billing link, keychain unlock, timeout extension).
- `AssistantToolErrorMessage` and `UserToolErrorMessage` both delegate to `RejectedPlanMessage` and `RejectedToolUseMessage` when a plan or tool use is explicitly rejected.
- Both use `safeParse` (Zod) validation as a safety net for corrupted or legacy-format data from resumed sessions.

### React Compiler Adoption
- Every component uses `react/compiler-runtime` (`_c(n)`) with a slot-based `$` cache array for manual memoization — this is the output of Babel's React Compiler. Each branch stores `{...props, component}` at specific indices so React can skip re-rendering unchanged sub-trees.
- Components range from 8 cache slots (`AssistantRedactedThinkingMessage`) to 49 (`UserTextMessage`) to 80 (`AssistantTextMessage`), reflecting the amount of branching logic.
- Sentinel values use `Symbol.for("react.memo_cache_sentinel")` for component-level cache entries.

### Terminal-Specific UI Patterns
- **Ink throughout**: All components use `ink` primitives (`Box`, `Text`, `Ansi`, `Newline`, `Spacer`, `useInput`, `useStdin`), not HTML.
- **`MessageResponse` wrapper**: The consistent single-line container for all messages, providing the `>`/chevron marker, optional top margin (`addMargin`), selected state background, and height control.
- **`dimColor` and `italic`** used for secondary/muted text; `color="success"` (green) for confirmations; `color="error"` (red) for errors; `color="suggestion"` (blue) for hints.
- **Windows Terminal fix**: `overflow="hidden"` is required on `RejectedPlanMessage`'s `Box` — without it, the box width is ignored on Windows.

### Feature Flags and Dead-Code Elimination
- `bun:bundle`'s `feature()` is used in `UserTextMessage` to conditionally import (via `require()`) components gated behind `KAIROS_GITHUB_WEBHOOKS`, `FORK_SUBAGENT`, `UDS_INBOX`, and `KAIROS`/`KAIROS_CHANNELS` — these are excluded from external builds entirely.
- The `EXTERNAL` feature flag gates the classifier approval system, removing it from the render tree in non-external builds.

### XML Tag Conventions
- **`teammate-message`**: Wraps teammate communications (`<teammate-message teammate_id="…" color="…" summary="…">…</teammate-message>`), parsed with `matchAll` against `TEAMMATE_MSG_REGEX`.
- **`bash-stdout`/`bash-stderr`/`local-command-stdout`/`local-command-stderr`**: Indicate styled output blocks within text.
- **`COMMAND_MESSAGE_TAG`** and **`TASK_NOTIFICATION_TAG`**: Signal command and task notification blocks.
- **`mcp-resource-update`/`mcp-polling-update`**: Signal MCP resource refresh events.
- **`fork-boilerplate`/`cross-session-message`/`channel`**: Feature-gated message types.

### Tool Rendering Contract
- Tool results delegate to the `Tool` object's own render methods (`renderToolResultMessage`, `renderToolUseErrorMessage`, `renderToolUseRejectedMessage`), making the tool rendering system extensible — new tools can define their own renderers without changing the routing layer.
- The `Tool` resolution path: `useGetToolFromMessages` looks up `lookups.toolUseByToolUseID` from the message store, then matches against `tools` by `name`, returning the `Tool` definition + `normalizedToolUse`.
- Post-tool hooks (e.g., Notion publish, Slack) are rendered as `HookProgressMessage` after successful tool results, loaded lazily via `require()` inside a `useEffect`.

### Filtering Silent Events
- `teammate_terminated`, `isShutdownApproved`, and `idle_notification` messages are **parsed** (so they don't leak into the output as garbage) but **filtered out** (no React node rendered) in both `UserTeammateMessage` and `AssistantTeammateMessage`.
- Empty messages (`isEmptyMessageText`) return `null` immediately in both `UserTextMessage` and `AssistantTextMessage`.
- `tick` tags and `LOCAL_COMMAND_CAVEAT_TAG` are also silently dropped in `UserTextMessage`.
