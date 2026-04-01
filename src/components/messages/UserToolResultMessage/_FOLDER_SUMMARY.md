# Summary of `components/messages/UserToolResultMessage/`

## Purpose of `UserToolResultMessage/`

This directory contains a **message rendering system** for displaying tool execution results in a CLI/terminal interface (built with [Ink.js](https://github.com/vadimdemedes/ink), React for CLIs). It handles the complete lifecycle of a tool's result: success, error, cancellation, and rejection — dispatching to specialized sub-components while providing validation, classifier approval badges, and post-tool hook execution.

## Contents Overview

| File | Role |
|------|------|
| **`index.tsx`** | Entry point / barrel re-export for the directory |
| **`UserToolResultMessage.tsx`** | **Router component** — reads `param.content` and result state, delegates to the appropriate child |
| **`UserToolSuccessMessage.tsx`** | Handles successful tool executions: renders output, classifier approval badges, post-tool hooks, validates output schema |
| **`UserToolErrorMessage.tsx`** | Handles failed tool executions: routes by content prefix to `InterruptedByUser`, `RejectedPlanMessage`, `RejectedToolUseMessage`, or a custom tool renderer |
| **`UserToolCanceledMessage.tsx`** | Thin wrapper rendering an `InterruptedByUser` component |
| **`UserToolRejectMessage.tsx`** | Renders rejected tool executions: validates input against schema, delegates to tool's `renderToolUseRejectedMessage` or falls back to `FallbackToolUseRejectedMessage` |
| **`RejectedToolUseMessage.tsx`** | Displays a dimmed "Tool use rejected" CLI message |
| **`RejectedPlanMessage.tsx`** | Displays a rejected plan in a styled bordered box with Markdown rendering |
| **`utils.tsx`** | Provides `useGetToolFromMessages` hook for resolving a tool definition + usage metadata from a `toolUseID` |

## How Files Relate to Each Other

```
UserToolResultMessage (router)
├── useGetToolFromMessages (utils.tsx)
│   └── lookups.toolUseByToolUseID + tools registry
│
├── UserToolCanceledMessage
│   └── InterruptedByUser
│
├── UserToolRejectMessage
│   ├── FallbackToolUseRejectedMessage
│   └── tool.renderToolUseRejectedMessage()
│
├── UserToolErrorMessage
│   ├── InterruptedByUser
│   ├── RejectedPlanMessage
│   ├── RejectedToolUseMessage
│   └── FallbackToolUseErrorMessage
│       └── tool.renderToolUseErrorMessage()
│
└── UserToolSuccessMessage
    ├── tool.renderToolResultMessage()
    ├── HookProgressMessage (post-tool hooks)
    └── SentryErrorBoundary
```

**Data flows top-down from the router:** `UserToolResultMessage` receives a `ToolResultBlockParam` (`param`) and `NormalizedUserMessage` from the message store, resolves the corresponding `Tool` via `useGetToolFromMessages`, inspects `param.content` (prefix-based dispatch) and `param.is_error` (boolean dispatch), and renders the matching child component — which then calls back into the `Tool` object's render methods (`renderToolResultMessage`, `renderToolUseErrorMessage`, etc.).

## Key Takeaways

- **Prefix-based routing** — The system uses string prefixes to identify special message types:
  - `CANCEL_MESSAGE` → canceled
  - `REJECT_MESSAGE` → rejected
  - `INTERRUPT_MESSAGE_FOR_TOOL_USE` → rejected
  - `PLAN_REJECTION_PREFIX` → plan rejection
  - `REJECT_MESSAGE_WITH_REASON_PREFIX` → tool use rejection
- **React Compiler** — All components use `react/compiler-runtime` with manual cache slot arrays (`$[n]`, `_c(n)`) for memoization, matching what the React Compiler would generate.
- **Schema validation as safety net** — `UserToolSuccessMessage` and `UserToolRejectMessage` validate tool I/O against Zod schemas (`safeParse`) to gracefully handle corrupted or legacy-format data from resumed sessions.
- **Compile-time feature flags** — Uses `bun:bundle`'s `feature()` for dead-code elimination; expensive subscriptions (classifier approvals, Sentry hooks) are conditionally compiled out in external builds.
- **CLI-specific design** — All UI components use Ink.js (`Box`, `Text`, `useTheme`), with `MessageResponse` as a consistent single-line container and `dimColor` for muted states. `overflow="hidden"` on `RejectedPlanMessage` is required for Windows Terminal compatibility.
- **Separation of concerns** — The router is stateless and purely dispatching; each sub-component handles its own data fetching (classifier approvals, tool rendering, hooks), and `utils.tsx` provides a single shared hook to decouple tool resolution from the rendering tree.
