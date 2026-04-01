# Summary of `services/`

# `services/` Directory

## Purpose

The `services/` directory is the **core runtime engine** of Claude Code, containing all backend logic for AI interactions, tool execution, audio I/O, telemetry, configuration management, and developer tooling. It bridges the gap between user-facing UI (terminal, IDE extension) and Anthropic's API, handling everything from session initialization to graceful shutdown.

---

## Contents Overview

| Subdirectory / File | Role |
|---------------------|------|
| **`analytics/`** | Multi-sink telemetry pipeline — routes events to Datadog and first-party OTel sink, with GrowthBook feature gates, killswitches, and PII sanitization |
| **`AgentSummary/`** | Periodic background summarization for coordinator-mode sub-agents — forks agent transcript on a 30s timer to produce UI progress labels |
| **`api/`** | Unified HTTP client layer for Anthropic's Claude API and Claude AI OAuth endpoints — retry logic, auth headers, bootstrap config, billing, usage, organization management |
| **`auditLog/`** | Structured audit trail for security-sensitive operations — appends signed `AuditEvent` records to a local encrypted SQLite database |
| **`autocomplete/`** | Shell completion generation and management for CLI commands, SSH config, and SSH host aliases |
| **`bash/`** | Bash/PowerShell command execution, input masking, history management, and TTY passthrough |
| **`beta/`** | Beta feature flag evaluation and messaging — gates experimental features behind user opt-ins |
| **`claudeAiLimits/`** | Rate limit detection and early-warning system — parses `ratelimit-*` headers, computes time-relative utilization, emits status changes to listeners |
| **`config/`** | Settings layer (`.claude/settings.json`) — JSON schema validation, dotfile migration, environment overrides, default value resolution |
| **`detect/`** | Project context detection — identifies git repos, package managers, programming languages, frameworks, and AI config files |
| **`diagnosticTracking/`** | LSP diagnostic capture and change detection — fetches baseline diagnostics, tracks new errors after edits, fetches IDE diagnostics on demand |
| **`feedback/`** | Anonymous crash/error reporting to Claude AI, with privacy controls and screenshot capture |
| **`fileExtensions/`** | Maps file extensions to language identifiers and syntax highlighting hints |
| **`fileReview/`** | Human-in-the-loop code review workflow — queues changes, assigns reviewers, handles approval/rejection/rounds |
| **`git/`** | Git operations — commit staging, status, diff, blame, branch operations, and path-aware git calls |
| **`growclust/`** | Clustering algorithm for grouping semantically related git commits into logical change clusters |
| **`httpUtils/`** | Shared HTTP utilities — JSON serialization, streaming helpers, redirect handling, TLS/mTLS config |
| **`ideProtocol/`** | IDE integration protocol — message framing, command routing, terminal buffer management |
| **`jvmUtils/`** | JVM process spawning — manages embedded Claude model processes with classpath and memory tuning |
| **`language/`** | Language detection and file classification — infers programming language from file extensions and content |
| **`markdown/`** | Markdown rendering and syntax highlighting for CLI and IDE display |
| **`memory/`** | Persistent memory stores — project context, session memory, conversation summaries, auto-context rules |
| **`mentions/`** | `@mention` resolution in prompts — maps workspace symbols, git commits, files, and command names to identifiers |
| **`mcp/`** | MCP (Model Context Protocol) client runtime — protocol framing, JSON-RPC 2.0 messaging, SSE transport, tool permission management |
| **`mcpUtils/`** | MCP server management — lifecycle (start/stop), port assignment, URI scheme routing, resource tracking |
| **`model/`** | Model selection, capability queries, and parameter normalization across Anthropic model families |
| **`nvm/`** | Node version management — installs and switches between Node.js versions for plugin compatibility |
| **`permissions/`** | Tool use permission system — permission types, user prompts, session persistence, trust shortcuts, IDE integration |
| **`preloadPlugins/`** | Plugin discovery and initialization — loads `.claude/plugins/` at startup, validates manifests, registers commands |
| **`rateLimitMocking/`** | Testing infrastructure for rate limiting — injects mock headers into HTTP responses |
| **`repoHealth/`** | Repository health checks — validates `.claude/` configuration, prompts for missing settings |
| **`security/`** | Security utilities — secret scanning, TLS hardening, path canonicalization |
| **`serviceTools/`** | Service-level tool implementations — memory, system, MCP, file operations |
| **`setup/`** | Interactive onboarding — model selection, authentication, plugin installation, first-run checks |
| **`speaker/`** | Text-to-speech output — converts model responses to audio using `say` or `espeak` |
| **`spinner/`** | Terminal spinner and progress indicators — customizable messages, progress bars, multi-line mode, keyboard shortcuts |
| **`status/`** | Statusline integration for VS Code and Zed — delegates to language-server-based implementations |
| **`systemPromptUtils/`** | System prompt assembly — builds role definitions, appends context hints, deduplicates content, formats agent instructions |
| **`taskHistory/`** | Persistent task history — appends completed task summaries to `~/.claude/tasks/` for recovery and resume |
| **`teamMemory/`** | Git-based team memory sync — watches `.claude/` directory, batches delta uploads to server, handles conflict resolution via hash probe |
| **`telemetry/`** | OpenTelemetry instrumentation — trace/span creation, attribute enrichment, SDK shutdown handling |
| **`tips/`** | Contextual tip display system — schedules tips by longest-idle algorithm, filters by platform/IDE, persists history to config |
| **`tools/`** | Tool execution pipeline — streaming executor, permission hooks, result buffering, concurrent vs serial orchestration |
| **`toolUseSummary/`** | AI-generated one-line summaries of tool execution batches using Haiku model |
| **`tunnel/`** | SSH tunnel management — connects to `ssh://` remotes via Claude Code Mobile or built-in forwarding |
| **`usage/`** | Usage metrics collection and reporting — token counting, cost estimation, session summarization |
| **`utils/`** | Shared utilities — ANSI colors, async helpers, diff computation, emoji, fs operations, logging, privacy |
| **`voice/`** | Audio recording — microphone input via native audio-capture-napi (macOS), ALSA arecord (Linux), or SoX rec (final fallback); voice keyterms for STT accuracy |
| **`voiceKeyterms/`** | Speech-to-text keyword boosting — builds domain-specific vocabulary list from project name, branch, and recent files |
| **`voiceStreamSTT/`** | WebSocket STT client for hold-to-talk — streams PCM audio to `voice_stream` endpoint, handles Cloudflare bypass, multi-timer finalize resolution |

---

## How Files Relate to Each Other

The services directory is organized around **data flow from user input to API response**, with several cross-cutting concerns layered horizontally.

### Primary Request Flow

```
User Input (terminal keystrokes, IDE LSP events, audio)
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  auth.js ─── authenticates request (OAuth or API key)        │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  claude.ts ─── executes LLM API calls (streaming or not)     │
│     ├── withRetry.ts ─── handles 401/429/529 retries       │
│     ├── model/ ─── selects model, builds request params     │
│     └── http.ts ─── assembles headers, handles 401 refresh │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  toolOrchestration.ts ─── partitions tool calls into batches│
│     ├── StreamingToolExecutor.ts ─── executes with streaming│
│     │     └── toolExecution.ts ─── single-tool runner      │
│     │           └── toolHooks.ts ─── permission hooks       │
│     ├── bash/ ─── executes shell commands                   │
│     ├── git/ ─── runs git operations                        │
│     └── mcp/ ─── forwards to MCP servers (mcpUtils/)        │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  memory/ ─── stores conversation, updates session context    │
│  auditLog/ ─── appends signed audit events                  │
│  analytics/ ─── emits telemetry to Datadog + 1P sink        │
└─────────────────────────────────────────────────────────────┘
```

### Cross-Cutting Concerns

| Concern | Service(s) | How it touches everything |
|---------|------------|---------------------------|
| **Rate limiting** | `claudeAiLimits.ts`, `claudeAiLimitsHook.ts` | Listens to API response headers; updates `currentLimits`; statusline displays warnings |
| **Permissions** | `permissions/` | Intercepts all tool execution via `toolHooks.ts`; checks trust rules before running any tool |
| **Configuration** | `config/`, `beta/` | All services read from `getGlobalConfig()` and feature flags; beta gates enable experimental features |
| **Error handling** | `feedback/`, `utils/log.js` | Errors propagate up through all layers; `feedback.ts` captures and reports crashes |
| **Security** | `security/`, `auditLog/` | Secret scanning in `toolHooks.ts`; audit trail for sensitive operations |
| **Telemetry** | `analytics/`, `telemetry/` | Every operation emits OTel spans; analytics enrich events with platform/model/process metadata |
| **Startup/shutdown** | `setup/`, `preloadPlugins/`, `repoHealth/` | Run once at startup; register commands, check config, install plugins |

### Audio Input Pipeline (Specialized Flow)

```
Microphone ──► voice/recording.ts (native/arecord/SoX)
                     │
                     ▼ (raw PCM)
          ┌──────────────────────────────────┐
          │  voiceStreamSTT.ts               │
          │  ┌────────────────────────────┐  │
          │  │ WebSocket to voice_stream  │  │
          │  └──────────────┬─────────────┘  │
          └────────────────┼────────────────┘
                           ▼
              ┌─────────────────────────────┐
              │  Transcripts via callbacks  │
              │  (onTranscript, onReady,     │
              │   onError, onClose)         │
              └────────────┬────────────────┘
                           ▼
              ┌─────────────────────────────┐
              │  voiceKeyterms.ts           │
              │  (boosts STT accuracy with  │
              │   project/branch keywords)  │
              └─────────────────────────────┘
```

### Team Memory Sync Flow

```
Watcher detects file change
        │
        ▼ (2s debounce)
┌──────────────────────────────────────────────┐
│  teamMemory/index.ts — pushTeamMemory()     │
│    ├─ readLocalFiles()                       │
│    ├─ diff against serverChecksums            │
│    ├─ batch into ≤200KB chunks               │
│    └─ PUT delta to server                    │
│         │                                    │
│         │ 412 Conflict?                      │
│         ▼                                    │
│    serverProbe() → retry (≤2×)               │
└──────────────────────────────────────────────┘
        │
        ▼ (on startup)
┌──────────────────────────────────────────────┐
│  pullTeamMemory() — writes server state      │
│  teamMemSecretGuard.ts — blocks secret writes│
└──────────────────────────────────────────────┘
```

---

## Key Takeaways

### 1. Fail-Open Design Everywhere
Analytics, feedback, telemetry, and secret scanning all fail silently — returning `null`, `false`, or empty arrays rather than throwing. Core functionality never depends on non-critical services succeeding.

### 2. Streaming is the Default
The entire request pipeline is built around streaming — `StreamingToolExecutor` buffers results in arrival order while yielding progress immediately. Even synchronous-looking functions like `generateToolUseSummary` are async for consistency.

### 3. Concurrency is Typed and Explicit
Tool calls declare `isConcurrencySafe`; the executor enforces serial execution for unsafe tools and parallel batching (up to 10 concurrent) for safe ones. Abort signals cascade through the hierarchy: user interrupt → Bash error → sibling tools.

### 4. Multi-Sink Analytics with Feature Gates
Every analytics event routes to Datadog (general-access) and first-party OTel (privileged-access) simultaneously, with GrowthBook controlling sampling rates, killswitches, and feature flags per-event-type. Proto field stripping ensures PII never reaches Datadog.

### 5. Security in Depth
Secret scanning happens at three layers: `secretScanner.ts` (client-side), `teamMemSecretGuard.ts` (file write guard), and `security/` utilities (TLS hardening). Audit logging provides a tamper-evident trail for sensitive operations.

### 6. Context-Aware Rate Limiting
`claudeAiLimits.ts` uses a two-stage early warning system — server-side `surpassed-threshold` headers when available, and a client-side time-relative fallback that warns when users burn quota faster than the window elapses.

### 7. Graceful Degradation for Audio
The audio pipeline chains three fallback strategies (native → ALSA → SoX) with environment-specific detection (WSL audio, remote sessions, macOS TCC). Voice STT bypasses Cloudflare by targeting `api.anthropic.com` directly.

### 8. Project Context is Everything
`detect/`, `memory/`, `teamMemory/`, `contextRules/`, and `mentions/` all work together to build the contextual backdrop for every LLM request — ensuring Claude understands the project, recent changes, team conventions, and user intent.
