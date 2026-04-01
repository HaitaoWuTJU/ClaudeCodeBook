# Summary of `commands/clear/`

## Purpose of `commands/clear/`

Implements the `/clear` command for a CLI tool, providing comprehensive conversation session reset functionality. This includes executing session end hooks, clearing caches, terminating foreground tasks while preserving background agents, regenerating session IDs, and executing session start hooks for the new session.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command entry point with metadata (`name`, `aliases`, `nonInteractive`) and lazy-loading implementation |
| `clear.ts` | Thin command handler that invokes `clearConversation()` and returns an empty response |
| `conversation.ts` | Core orchestration logic (~9.3KB) handling the full reset workflow |
| `caches.ts` | Collection of cache-clearing utilities (~1.7KB) used by both `clear.ts` and `conversation.ts` |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│ index.ts                                                        │
│  - Loads on startup (minimal imports)                           │
│  - Lazy-loads ./clear.js when command is invoked                │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ clear.ts                                                        │
│  - Imports clearConversation from ./conversation.ts            │
│  - Returns { type: 'text', value: '' }                          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
┌──────────────────────────┐  ┌─────────────────────────────────┐
│ caches.ts                │  │ conversation.ts                 │
│                          │  │                                 │
│ clearSessionCaches() ◄───┼──┤─► Imports clearSessionCaches()  │
│  - context caches        │  │  - Executes session hooks       │
│  - file suggestion cache │  │  - Kills foreground tasks       │
│  - commands cache        │  │  - Preserves background agents   │
│  - session ingress       │  │  - Regenerates session ID        │
│  - post-compact cleanup  │  │  - Executes session start hooks  │
│  - LSP diagnostic state  │  │                                 │
│  - dynamic skills       │  │                                 │
│  - conditional: Tungsten │  │                                 │
│  - conditional: WebFetch│  │                                 │
│  - conditional: ToolSearch│                                 │
│  - conditional: Agents   │  │                                 │
│  - conditional: SkillTool│  │                                 │
│  - conditional: attribution│                               │
└──────────────────────────┘  └─────────────────────────────────┘
```

## Key Takeaways

1. **Lazy-loading optimization** — `index.ts` defers all heavy imports until the command executes, keeping startup lightweight.

2. **Task preservation strategy** — Background agents and tasks spawned via Ctrl+B survive `/clear`; only foreground tasks are terminated. Per-agent state (invoked skills, pending callbacks) is retained for preserved tasks.

3. **Analytics lineage** — Session IDs regenerate with parent lineage tracking, enabling analytics to trace conversations across multiple `/clear` operations.

4. **Comprehensive cleanup** — `caches.ts` clears 20+ separate cache regions across file discovery, prompts, MCP state, memory, and tool descriptions.

5. **Feature-gated imports** — Conditional dynamic imports (Tungsten, WebFetch, ToolSearch, Agents, SkillTool, attribution) preserve dead code elimination for disabled features.

6. **Hook lifecycle** — Session end hooks run first (bounded by timeout), then state clears, then session start hooks run to completion before returning.
