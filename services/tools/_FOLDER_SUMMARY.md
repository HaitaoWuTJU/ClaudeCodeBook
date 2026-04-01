# Summary of `services/tools/`

## Purpose of `tools/`

The `tools/` directory implements the complete tool execution pipeline for an AI coding assistant, enabling Claude to invoke external tools (bash commands, file operations, search, MCP integrations) with full streaming support, permission management, telemetry, and extensibility through hooks.

## Contents Overview

| File | Responsibility |
|------|-----------------|
| `StreamingToolExecutor.ts` | Manages streaming arrival of tool calls; handles concurrent vs serial execution, abort cascades, result buffering, and progress yielding |
| `toolExecution.ts` | Core single-tool execution engine: permission resolution, hook invocation, result processing, telemetry logging |
| `toolHooks.ts` | Implements PreToolUse, PostToolUse, and PostToolUseFailure hooks with permission decision resolution |
| `toolOrchestration.ts` | Top-level orchestrator that partitions tool calls into batches and coordinates the execution pipeline |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────────┐
│                      toolOrchestration.ts                            │
│  Entry point — partitions tool calls, runs batches sequentially      │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ ToolUseBlock[]
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    StreamingToolExecutor.ts                         │
│  Queues tools, manages concurrency (parallel for safe, serial for   │
│  exclusive), handles abort propagation to siblings, yields progress  │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ Single tool execution
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       toolExecution.ts                               │
│  Resolves permissions, runs pre-hooks, executes tool, runs post-    │
│  hooks, logs telemetry, yields result messages                       │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ Hook execution
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        toolHooks.ts                                  │
│  Executes PreToolUse, PostToolUse, PostToolUseFailure hooks;        │
│  resolves hook permission decisions against settings rules          │
└──────────────────────────────────────────────────────────────────────┘
```

**Call hierarchy** (outer → inner):

1. `runTools()` from `toolOrchestration.ts` accepts raw tool blocks from a message
2. It creates a `StreamingToolExecutor` and adds each `ToolUseBlock`
3. `StreamingToolExecutor.addTool()` checks concurrency safety via `findToolByName` and starts execution when possible
4. `StreamingToolExecutor.executeTool()` calls `runToolUse()` from `toolExecution.ts`
5. `runToolUse()` invokes `executePreToolHooks()` and `executePostToolHooks()` from `toolHooks.ts`
6. Results flow back through the chain, yielding `MessageUpdate` objects with messages and updated context

## Key Takeaways

- **Concurrency is opt-in per tool** — tools declare `isConcurrencySafe`; safe tools run in parallel (up to `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`, default 10), while non-safe tools run exclusively one at a time

- **Abort propagation is multi-layered** — user interrupts flow through a parent controller; Bash errors cascade to siblings via a dedicated child controller; per-tool controllers handle permission dialog rejections

- **Hooks can intercept and modify the execution pipeline** — PreToolUse hooks can block execution, update inputs, or grant permissions; PostToolUse hooks can emit attachments and update MCP outputs

- **Results are yielded in arrival order** — `StreamingToolExecutor` buffers completed results and emits them in the order tools were queued, while progress messages bypass buffering for immediate delivery

- **Telemetry is comprehensive** — every execution phase emits OTel spans and analytics events (`tengu_tool_use_*`, `tengu_pre/post_tool_hook_*`) with sanitized tool names and git commit metadata

- **MCP tools are first-class citizens** — detected by `mcp__` prefix and `isMcpTool()`; errors include auth-specific handling (`McpAuthError`); post-hook flows are adapted for MCP output semantics
