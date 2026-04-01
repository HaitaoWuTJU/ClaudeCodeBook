# Summary of `./`

## Purpose of `src/`

The `src/` directory is the **primary codebase** for Claude Code — a CLI agent that uses AI models to assist with software development tasks. It implements a full development workflow engine with:

- **Slash command system** (`commands/`) for user-facing CLI commands
- **Tool system** (`tools/`, `tools.ts`) for model-invokable capabilities
- **Vim emulation** (`vim/`) for modal text editing in the embedded REPL
- **Remote control bridge** (`bridge/`) for cloud-connected sessions
- **Companion system** (`buddy/`) for mascot-based UX
- **Core infrastructure** (`bootstrap/`, `utils/`, `analytics/`, `telemetry/`)
- **Context management** (`context.ts`) for prompt injection
- **Cost tracking** (`cost-tracker.ts`) for API usage monitoring

## Contents Overview

| Directory/File | Primary Role |
|----------------|--------------|
| **`commands/`** | 40+ slash command implementations (add, autofixPr, bash, config, diff, git, etc.) |
| **`commands.ts`** | Central command registry — aggregates all built-in, skill, plugin, and dynamic commands with feature-gated loading and permission filtering |
| **`context.ts`** | Conversation context preparation — builds system context (git status) and user context (Claude.md files) with memoization |
| **`cost-tracker.ts`** | API cost and usage tracking — accumulates per-model metrics, persists to project config, formats reports |
| **`index.ts`** | Main entry point — orchestrates initialization, session resume, REPL/agent loop, and exit |
| **`tools/`** | 25+ tool implementations (BashTool, FileReadTool, AgentTool, WebSearchTool, etc.) |
| **`tools.ts`** | Central tool registry — assembles tool pools, filters by permissions, handles MCP tools with deduplication |
| **`assistant/`** | Teleport Agent SDK — API client with OAuth support, session history retrieval, browser/Node.js dual runtime |
| **`bootstrap/`** | App initialization — `state.ts` provides the global `STATE` singleton consumed by all modules |
| **`bridge/`** | Remote control infrastructure — environment registration, work polling, OAuth, trusted device tokens, heartbeat |
| **`buddy/`** | Companion mascot — procedural ASCII art generation, React sprite component, AI prompt utilities |
| **`utils/`** | ~40 subdirectories of shared utilities: bash parsing/security, git operations, settings management, telemetry, analytics, error handling |
| **`vim/`** | Vim emulation layer — 5-module pure state machine for NORMAL/INSERT modes, motions, operators, text objects |
| **`voice/`** | Voice mode feature-gating — combines GrowthBook flags and OAuth validation |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              ENTRY POINT                                │
│   index.ts (main)                                                        │
│       │                                                                  │
│       ▼                                                                  │
│   ┌─────────────────────────┐    ┌──────────────────────────────────┐   │
│   │   bootstrap/state.ts    │◄───│  Telemetry providers (OpenTelemetry) │
│   │   (STATE singleton)     │    └──────────────────────────────────┘   │
│   └───────────┬─────────────┘                                             │
│               │                                                           │
│               ▼                                                           │
│   ┌───────────────────────────────────────────────────────────────────┐   │
│   │                      REGISTRY LAYER                               │   │
│   │                                                                     │   │
│   │   commands.ts ◄── getCommands(cwd)                                │   │
│   │       │           ├──► Skill loaders                               │   │
│   │       │           ├──► Plugin commands                             │   │
│   │       │           └──► Workflow commands                            │   │
│   │       │                                                            │   │
│   │   tools.ts ──► getTools(permissionContext)                         │   │
│   │       │           ├──► MCP tools (deduplicated)                    │   │
│   │       │           ├──► REPL-only tools (hidden in REPL mode)         │   │
│   │       │           └──► Feature-gated tools                          │   │
│   │       │                                                            │   │
│   │   cost-tracker.ts ── addToTotalSessionCost()                       │   │
│   │       │              ├──► addToTotalModelUsage()                    │   │
│   │       │              └──► Project config persistence                 │   │
│   │       │                                                            │   │
│   │   context.ts ──► getUserContext() / getSystemContext()             │   │
│   │                   ├──► Git status (execFileNoThrow)                 │   │
│   │                   └──► Claude.md files (fs reads)                  │   │
│   └───────────────────────────────────────────────────────────────────┘   │
│               │                                                           │
│               ▼                                                           │
│   ┌───────────────────────────────────────────────────────────────────┐   │
│   │                    DOMAIN CLUSTERS                                │   │
│   │                                                                     │   │
│   │   bash/ ──► exec.ts ──► ShellSnapshot.ts ──► Security checks       │   │
│   │       │       │                                                   │   │
│   │       │       └──► parser.ts (tree-sitter) / bashParser.ts        │   │
│   │       │               └──► ast.ts (validation)                     │   │
│   │       │                                                            │   │
│   │   git/  ──► git.js ──► gitBundle.ts ──► git.ts (repo detection)  │   │
│   │       │                                                            │   │
│   │   settings/ ──► settings.ts ──► project config persistence        │   │
│   │       │                                                            │   │
│   │   analytics/ ──► sessionAnalytics.ts ──► bigqueryExporter.ts    │   │
│   │       │                                                            │   │
│   │   telemetry/ ──► sessionTracing.ts ──► events.ts                  │   │
│   │       │                          └──► perfettoTracing.ts          │   │
│   │       │                                                            │   │
│   │   teleport/ ──► api.ts ──► OAuth + API requests                   │   │
│   │       │                                                            │   │
│   │   bridge/ ──► bridgeApi.ts ──► bridgeConfig.ts                     │   │
│   │       │                                                            │   │
│   │   vim/  ──► transitions.ts ──► motions.ts, operators.ts, ...      │   │
│   │       │                                                            │   │
│   │   buddy/ ──► companion.js ──► sprites.ts ──► CompanionSprite.tsx   │   │
│   │       │                                                            │   │
│   │   voice/ ──► voice-mode.ts ──► growthbook.js + auth.js            │   │
│   └───────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Primary Initialization Sequence

```
index.ts
  ├── initTelemetry() ──► bootstrap/state.ts (STATE.telemetry)
  │         └──► OpenTelemetry Meter/Logger/Tracer providers
  ├── initCommandCache() ──► commands.ts (COMMANDS cache)
  ├── initToolCache() ──► tools.ts (assembleToolPool cache)
  └── runSession() ──► context.ts (getSystemContext, getUserContext)
              └──► Session analytics + tracing
```

### Command Execution Path

```
User input (slash command)
  └── commands.ts (findCommand)
              ├──► meetsAvailabilityRequirement() ──► auth.js
              ├──► isEnabled() ──► feature() flags
              └──► Tool handler ──► tools/ + STATE hooks
```

## Key Takeaways

### 1. Layered Architecture with Clear Boundaries

The codebase is organized into **five layers** with strictly directional dependencies:

| Layer | Contents | Rule |
|-------|----------|------|
| **Utilities** | `utils/` (errors, log, fetch, fs) | No imports from domain clusters |
| **Domain Clusters** | `bash/`, `git/`, `teleport/`, `analytics/`, `settings/` | Imports only utilities, no cross-cluster deps |
| **Registries** | `commands.ts`, `tools.ts`, `cost-tracker.ts` | Import domain clusters and utilities |
| **State** | `bootstrap/state.ts` | Imports utilities only, exported as singleton |
| **Entry** | `index.ts` | Orchestrates all layers |

This isolation enables independent evolution of each domain and simplifies testing.

### 2. Feature Gating via `bun:bundle`

`feature()` from `bun:bundle` is the primary mechanism for **dead-code elimination**. Conditional imports wrapped with feature checks (`VOICE_MODE`, `WORKFLOW_SCRIPTS`, `KAIROS`, etc.) are tree-shaken in builds where those features are disabled. This appears in `commands.ts`, `tools.ts`, and `utils/` feature-gated modules.

### 3. Dual Memoization Patterns

Two distinct caching strategies coexist:

- **`lodash-es/memoize`**: Used in `commands.ts`, `context.ts` for **explicit memoization** with typed arguments. `clearCommandsCache()` provides manual invalidation for dynamic skill/plugin discovery.
- **`AsyncLocalStorage`** + module-level maps: Used in `analytics/` and `telemetry/` for **context propagation** without explicit threading. Enables per-subagent attribution without changing function signatures.

### 4. Fail-Closed Security Model

The `bash/` cluster implements a **strict fail-closed posture**: any command feature that cannot be statically verified is rejected (`kind: 'too-complex'`). This includes command substitutions, process substitutions, parameter expansions, and compound statements. The codebase prefers rejecting ambiguous commands over silently executing them. `bash/security.ts` is the enforcement point for all command execution.

### 5. Context Is King

Every AI interaction flows through `context.ts`, which prepends:

- **System context**: Git status (branch, recent commits, user), cache breaker injection
- **User context**: Claude.md content, current date

Both are memoized per session, and cache invalidation is coordinated via `setSystemPromptInjection()` in ant-mode. This ensures consistent AI behavior within a session.

### 6. Remote/Bridge as First-Class Mode

`bridge/` is not an afterthought — it's a fully-layered subsystem:

- `bridgeConfig` provides environment-specific configuration
- `bridgeApi` handles all REST API communication with retry logic
- `workSecret` decodes and routes to WebSocket or REST based on session type
- `bridgeDebug` enables fault injection for testing recovery paths

### 7. Telemetry-First Design

Every significant operation feeds into the OpenTelemetry pipeline:

- `sessionTracing.ts` creates spans for commands, tools, and model calls
- `events.ts` enriches spans with contextual metadata
- `bigqueryExporter.ts` persists to BigQuery asynchronously
- `perfettoTracing.ts` provides parallel local trace output

Feature flags (`OTEL_LOG_USER_PROMPTS`, `BETA_SESSION_TRACING`) control verbosity without removing code paths.

### 8. Vim as a Pure State Machine

The `vim/` directory implements Vim's modal editing entirely as **pure functions** below the state machine layer. `transitions.ts` is the only module with side effects (via `TransitionContext` callbacks). This makes the ~650 lines of motion, operator, and text-object logic trivially testable — the entire Vim implementation has no test file because its purity guarantees correctness.

### 9. Dot-Repeat via Discriminated Union

Vim's `.` repeat captures every repeatable command as a `RecordedChange` discriminated union with 11 variants in `vim/types.ts`. This enables accurate replay for complex operations like `ci(`, `dG`, `>>`, `p`, not just simple single-keystroke motions.

### 10. Cost Tracking as a First-Class Feature

`cost-tracker.ts` is not an afterthought — it's integrated at every model call:

- Tracks per-model token counts, API costs, and session duration
- Persists to project `.claude/` config for session resume
- Formats comprehensive usage reports with model-level breakdowns
- Used for both user-visible output and potential billing attribution
