# Summary of `tools/AgentTool/`

## Purpose of `AgentTool/`

The `AgentTool/` directory is the orchestration core for sub-agent execution in Claude Code. It manages the complete lifecycle of agent spawning—from memory and tool resolution through execution, UI rendering, and cleanup—while providing both a generic `AgentTool` interface and a collection of built-in specialized agents for common tasks.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.ts` | Re-exports public API and registers the `AgentTool` |
| `AgentTool.ts` | Defines input/output schemas (`inputSchema`, `outputSchema`), `Progress` union, and tool initialization |
| `agentColorManager.ts` | Maps agent types to theme colors for visual differentiation in UI |
| `agentDisplay.ts` | Resolves agent overrides, builds display strings, and provides sorting utilities |
| `agentMemory.ts` | Manages persistent `MEMORY.md` directories across three scopes (user/project/local) |
| `agentMemorySnapshot.ts` | Handles snapshot checking, initialization, and sync metadata for agent memory |
| `runAgent.ts` | Async generator orchestrating agent execution: context building, MCP setup, tool resolution, message streaming, and cleanup |
| `UI.tsx` | React/CLI UI rendering progress messages, tool results, transcripts, and error states |
| `built-in/` | Six pre-configured agents: `explore`, `general-purpose`, `plan`, `claude-code-guide`, `statusline-setup`, `verification` |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              AgentTool (entry point)                          │
│                         inputSchema + outputSchema + Progress                 │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │ spawned by orchestrator
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                 runAgent.ts                                    │
│  ┌────────────────┐  ┌────────────────┐  ┌─────────────┐  ┌────────────────┐   │
│  │ MCP servers    │  │ Tool resolution│  │ Context     │  │ Message stream │   │
│  │ (initialize)   │  │ (agentToolUtils)│ │ building    │  │ (yield loop)   │   │
│  └───────┬────────┘  └───────┬────────┘  └──────┬──────┘  └───────┬────────┘   │
└──────────┼───────────────────┼─────────────────┼──────────────────┼───────────┘
           │                   │                 │                  │
           ▼                   ▼                 ▼                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Supporting Modules                                   │
│  ┌─────────────────┐  ┌────────────────┐  ┌─────────────────┐  ┌────────────┐  │
│  │ agentColorManager│  │ agentDisplay  │  │ agentMemory     │  │ built-in/  │  │
│  │ (visual theming)│  │ (override rslv)│ │ (persistence)   │  │ (agents)   │  │
│  └────────┬────────┘  └───────┬────────┘  └────────┬────────┘  └─────┬──────┘  │
│           │                   │                   │                │         │
└───────────┼───────────────────┼───────────────────┼────────────────┼─────────┘
            │                   │                   │                │
            ▼                   ▼                   ▼                ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                              UI.tsx                                           │
│  ┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────────┐      │
│  │ AgentResponseDisplay │  │ VerboseTranscript│  │ renderToolResultMsg │      │
│  │ (content blocks)    │  │ (progress lines) │  │ (status rendering)  │      │
│  └──────────────────────┘  └──────────────────┘  └──────────────────────┘      │
└────────────────────────────────────────────────────────────────────────────┘
```

**Data flow for a typical agent invocation:**

1. Orchestrator calls `runAgent()` with an `AgentDefinition` from `loadAgentsDir.js`
2. `runAgent()` resolves tools via `agentToolUtils.js`, initializes MCP servers, and builds context
3. `agentMemory.ts` is consulted (via session hooks or `getAgentSystemPrompt()`) for any `MEMORY.md` content
4. Message stream yields to `UI.tsx` via `ProgressMessage<Progress>` events
5. `agentDisplay.ts` merges `allAgents` against `activeAgents` to detect override chains
6. `agentColorManager.ts` provides theme color keys for the UI to render agent badges
7. Cleanup in `runAgent()` finally block releases all resources

## Key Takeaways

1. **Agent orchestration is fully async/generator-based**: `runAgent` is an `AsyncGenerator<Message>` that yields progress events for streaming UI updates, enabling real-time display of tool calls, search operations, and completion summaries.

2. **MCP servers are first-class citizens**: Agent definitions can declare MCP servers inline (scoped as `'dynamic'`, cleaned up after execution) or reference named configs (memoized, shared with parent). This enables per-agent tool augmentation without global side effects.

3. **Memory is scoped and snapshot-aware**: Three memory scopes (user/project/local) with dedicated directories, path normalization for traversal protection, and optional snapshot initialization for first-time agents or remote setups.

4. **Override chains are resolved at display time**: `agentDisplay.ts` compares `allAgents` against `activeAgents` using `agentType` + `source` keys, supporting git worktree scenarios where duplicate agent definitions exist across paths.

5. **Built-in agents follow a read-only specialization pattern**: Five of six built-in agents (`explore`, `plan`, `claude-code-guide`, `verification`, `statusline-setup`) are strictly read-only. `generalPurposeAgent` is the sole agent that can modify files, though it explicitly blocks sub-agent spawning.

6. **UI rendering is decoupled from execution**: `UI.tsx` receives only serialized `ProgressMessage` events and transforms them into terminal-friendly React/Ink components, grouping consecutive search/read operations into summary messages to reduce noise.

7. **Color and theme are context-driven**: `agentColorManager.ts` maps agent types to theme color keys (`*_FOR_SUBAGENTS_ONLY`), allowing visual differentiation in the CLI without hardcoding hex values. The `'general-purpose'` agent is excluded from coloring.

8. **Verification uses structured output**: The verification agent outputs a `VERDICT: PASS|PARTIAL|FAIL` prefix, making it machine-parseable for CI integration, while restricting all writes to `/tmp` only.
