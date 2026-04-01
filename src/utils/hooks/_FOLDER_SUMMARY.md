# Summary of `utils/hooks/`

## Purpose of `utils/hooks/`

This directory implements the **hook system** for Claude Code — a plugin architecture that allows custom code to run at specific points during agent execution (before/after LLM queries, on tool results, after turns, on session start/end, etc.). The system supports both synchronous hooks (command-line scripts, prompt modifications) and asynchronous hooks (shell commands that respond via stdout), with safeguards for security (SSRF protection), persistence (settings serialization), and session isolation.

## Contents Overview

| File | Responsibility |
|------|---------------|
| `apiQueryHookHelper.ts` | **Factory** for creating reusable API query hooks; handles LLM calls, response parsing, logging, and error normalization |
| `AsyncHookRegistry.ts` | **Async hook lifecycle manager** — tracks pending hooks by `processId`, polls shell commands for stdout JSON responses, manages 15s timeouts and progress intervals |
| `execAgentHook.ts` | **Agent-based hook executor** — spawns a sub-agent with tools to verify stop conditions, supports structured output, timeout/abort handling, and comprehensive analytics |
| `hooksSettings.ts` | **Persistence layer** — serializes/deserializes hook configurations to `settings.json`, deduplicates entries, filters dangerous patterns, handles skill-scoped hooks with project root resolution |
| `sessionHooks.ts` | **Session state manager** — stores hooks per session in a `Map` with in-place mutation (O(1) updates) to avoid triggering all store listeners during parallel agent execution |
| `skillImprovement.ts` | **Skill self-improvement** — batches every 5 user turns, analyzes conversation via small fast model, suggests/appends improvements to `.claude/skills/*/SKILL.md` |
| `ssrfGuard.ts` | **Security guard** — validates resolved IPs from DNS lookups against private/link-local ranges (including IPv4-mapped IPv6), blocks cloud metadata endpoints |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Hook Execution Flow                              │
└─────────────────────────────────────────────────────────────────────────┘

   ┌─────────────────────┐      ┌──────────────────────────────────────┐
   │   hooksSettings.ts  │      │        sessionHooks.ts               │
   │   (persistence)     │      │   (session state: Map<sessionId, …>) │
   └──────────┬──────────┘      └─────────────────┬────────────────────┘
              │                                       │
              │ load/save                            │ add/remove/get hooks
              ▼                                       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Hook Lifecycle                                     │
│                                                                          │
│  ┌─────────────────┐     ┌─────────────────────────────────────────┐   │
│  │ execAgentHook   │     │         AsyncHookRegistry                │   │
│  │ (agent-based    │     │  (async: shell stdout → JSON → emit)     │   │
│  │  stop condition │     └─────────────────────────────────────────┘   │
│  │  verification)  │                     ▲                               │
│  └─────────────────┘                     │ finalizeHook()               │
│          │                    ┌───────────┴───────────┐                 │
│          │                    ▼                       ▼                 │
│  ┌─────────────────┐  ┌──────────────┐        ┌──────────────┐         │
│  │ apiQueryHook    │  │ checkForAsync │        │   (shell     │         │
│  │ Helper         │  │ HookResponses │───────▶│  commands)   │         │
│  │ (LLM query +    │◀─┤  (polls all   │        │              │         │
│  │  parse + log)   │  │   pending)    │        └──────────────┘         │
│  └─────────────────┘  └──────────────┘                                   │
│          │                                                                │
│          │ LLM call (blocked by SSRF guard)                              │
└──────────┼────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────┐
│     ssrfGuard.ts    │
│ (IP range validation│
│  on axios lookup)   │
└─────────────────────┘
```

### Dependency Graph

```
apiQueryHookHelper.ts
├── imports: queryModelWithoutStreaming, createAbortController, getSmallFastModel, logError, toError
└── provides: createApiQueryHook() used by execAgentHook.ts, skillImprovement.ts

execAgentHook.ts
├── imports: query(), createStructuredOutputTool(), registerStructuredOutputEnforcement()
├── uses: createApiQueryHook (from apiQueryHookHelper)
└── provides: execAgentHook() used by postSamplingHooks.js

AsyncHookRegistry.ts
├── imports: ShellCommand, emitHookResponse(), startHookProgressInterval()
├── manages: Shell command lifecycle for async hooks
└── used by: postSamplingHooks.js (checkForAsyncHookResponses)

hooksSettings.ts
├── imports: SessionStore, AggregatedHookResult
├── reads/writes: settings.json via settings.js
└── provides: loadHooks(), saveHooks() used by sessionHooks.ts

sessionHooks.ts
├── imports: SessionStore, AppState, getSessionHooks(), getSessionFunctionHooks()
├── uses: hooksSettings.ts for persistence, appState for session-scoped state
└── provides: addSessionHook(), getSessionHooks() used by hook dispatchers

skillImprovement.ts
├── imports: registerPostSamplingHook(), createApiQueryHook
├── uses: apiQueryHookHelper for batch analysis + applySkillImprovement
└── modifies: .claude/skills/*/SKILL.md files

ssrfGuard.ts
├── exports: ssrfGuardedLookup() used as axios lookup option
└── blocks: requests to private/cloud-metadata IPs during HTTP hook execution
```

## Key Takeaways

### 1. Hook Types Are Decoupled from Lifecycle Management
Hooks are defined generically (`HookCommand`, `FunctionHook`) but their execution is handled by different mechanisms:
- **Synchronous hooks**: Run inline via `postSamplingHooks.js` — command strings, prompt modifiers, function callbacks
- **Async hooks**: Tracked in `AsyncHookRegistry.ts` — shell commands that send responses via stdout JSON; the registry polls for completion and emits responses after `TURN_BATCH_SIZE` batches

### 2. Performance-Sensitive Design: In-Place Mutation for Parallelism
`sessiohooks.ts` uses **in-place mutation** on a `Map` rather than immutable updates. This is critical because `parallel()` workflows can spawn N agents simultaneously, each calling `addFunctionHook()`. The `store.ts` equality check (`Object.is(next, prev)`) short-circuits all ~30 store listeners when the same `Map` reference is returned, avoiding an O(N²) listener storm.

### 3. Three-Layer Persistence Model
| Layer | Mechanism | Scope |
|-------|-----------|-------|
| `hooksSettings.ts` | `settings.json` (JSON) | Project-level, skill-scoped hooks |
| `sessionHooks.ts` | In-memory `Map<sessionId, SessionStore>` | Session-level, ephemeral |
| `postSamplingHooks.js` | `hookRegistry` global | Process-level, all sessions |

### 4. SSRF Protection Is Tightly Integrated
`ssrfGuard.ts` exports `ssrfGuardedLookup`, which is passed as the `lookup` option to axios. The guard intercepts **both** direct IP literals (via `net.isIP()`) and DNS-resolved addresses, blocking:
- Cloud metadata IPs (`169.254.0.0/16`) including IPv4-mapped forms (`::ffff:a9fe:a9fe`)
- All RFC 1918 private ranges
- IPv6 unique-local and link-local ranges

### 5. Agent Hooks Use Structured Output Enforcement
`execAgentHook.ts` creates a synthetic output tool (`__StructuredOutput__`) and registers a session-level enforcement hook to parse LLM responses for structured data, with 50 max turns and a small fast model for speed.

### 6. Error Handling Philosophy
- **apiQueryHookHelper**: Catches parse errors internally, reports via `logResult` without rethrowing — preserves agent flow
- **AsyncHookRegistry**: Uses `Promise.allSettled` to prevent one throwing callback from orphaning side effects on other hooks
- **execAgentHook**: Separates `error` (TypeScript/hook errors), `non_blocking_error` (LLM errors), and `blocking_error` (critical failures) outcome types

### 7. Skill Improvement Is a Fire-and-Forget Enhancement
The system batches every 5 user turns, runs a lightweight analysis on a fast model, stores suggestions in app state, and applies improvements asynchronously — never blocking the main conversation loop.
