# Summary of `tools/SendMessageTool/`

## Purpose of `SendMessageTool/`

Implements a swarm communication tool that allows agents in a multi-agent system to send messages to teammates, other processes, or remote sessions. It supports plain text messages, broadcasts, and structured protocol messages for shutdown and plan approval workflows.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Exports the canonical tool name constant `SEND_MESSAGE_TOOL_NAME` for use across the codebase |
| `prompt.ts` | Defines the system prompt (tool description, usage examples, protocol documentation) with feature-gated sections for Unix Domain Socket (UDS) and bridge peer support |
| `SendMessageTool.ts` | Core implementation — Zod schemas, routing logic, per-message-type handlers, permission checks, and Ink UI integration |
| `UI.tsx` | Ink-based terminal rendering functions for displaying plan approval prompts and result messages |

## How Files Relate to Each Other

```
index.ts
  └─► Exports SEND_MESSAGE_TOOL_NAME ("SendMessage")
           ▲
           │ (referenced by)
           │
prompt.ts ─┼─► Provides usage docs & examples used by SendMessageTool.ts
  │        │
  │        └─► References UDS_INBOX feature flag (bun:bundle)
  │
  └──► Consumed at runtime by the tool runner to render help text
              │
              ▼
SendMessageTool.ts ◄─── Implements the tool
  │                  ◄─── Uses Zod schemas from inputSchema/StructuredMessage
  │                  ◄─── Imports writeToMailbox, getAgentName, etc.
  │                  ◄─── Routes via UDS, bridge, mailbox, or in-process
  │                  ◄─── Calls gracefulShutdown, resumeAgentBackground, etc.
  │
  └──► UI.tsx ◄─── Exports renderToolUseMessage() and renderToolResultMessage()
                    ◄─── Imports Input & SendMessageToolOutput types
```

## Key Takeaways

- **Multi-channel routing** — The tool dispatches messages through four paths: mailbox (standard teammates), UDS socket (local peers), bridge (cross-session remote agents), or in-process agent resumption
- **Structured protocol** — Supports shutdown requests/responses and plan approval/rejection messages as discriminated union types
- **Security boundaries** — Cross-machine (bridge) messages require explicit user consent; the tool checks `isAgentSwarmsEnabled()` and validates permissions before sending
- **Feature-gated capabilities** — UDS and bridge peer support are controlled by `feature('UDS_INBOX')`, allowing selective rollout
- **In-process agent lifecycle** — In-process teammates can be auto-resumed when a message arrives; shutdown approvals trigger `abortController.abort()` or graceful shutdown
- **Ink UI** — The tool renders plan approval prompts and result messages using Ink (React for CLIs), not a web framework
