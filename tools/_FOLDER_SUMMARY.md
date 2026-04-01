# Summary of `tools/`

## Purpose of `tools/`

The `tools/` directory is the **tool layer** of Claude Code—a collection of composable, schema-driven utilities that give the AI model capabilities beyond plain text generation. Every tool follows a common interface contract (input validation, permissions, execution, UI rendering) and is registered with the orchestrator at startup. Tools are organized by domain into three categories: **filesystem operations**, **interactive/messaging**, and **external integrations**.

## Contents Overview

### Filesystem Operations

| Tool | Files | Purpose |
|------|-------|---------|
| **ReadTool** | `ReadTool/`, `ReadMultipleTool/` | Reads file content, respects `.claudeignore`, supports line ranges, binary detection, symlink following, multi-file reads with glob patterns |
| **GlobTool** | `GlobTool/` | Pattern-based file discovery with `.claudeignore` support, caching, sorted deterministic output |
| **GrepTool** | `GrepTool/` | Multi-pattern regex/text search with context lines, file-type filtering, ignore-file support, JSON output |
| **BashTool** | `BashTool/` | Shell command execution with PTY, permission management, security validation, path constraints, output truncation, LRU image cache |

### Interactive & Messaging

| Tool | Files | Purpose |
|------|-------|---------|
| **AskUserQuestionTool** | `AskUserQuestionTool/` | Presents 1–4 multiple-choice questions interactively in the terminal, returns structured answers with optional HTML/Markdown preview per option |
| **ExitPlanModeTool** | `ExitPlanModeTool/` | Exits plan-review mode, unlocking file modifications and other restricted operations |
| **BriefTool** | `BriefTool/` | Displays a project summary from `~/.claude/brief.md`, providing context without opening full memory |

### External Integrations

| Tool | Files | Purpose |
|------|-------|---------|
| **WebSearchTool** | `WebSearchTool/` | Streams web search results via Anthropic API, renders progress and formatted source links |
| **WebFetchTool** | `WebFetchTool/` | Fetches URL content, converts HTML to Markdown via turndown, uses secondary AI model for prompt-guided extraction |
| **AgentTool** | `AgentTool/` | Spawns sub-agents with configurable memory, MCP server isolation, and async message streaming |
| **ToolSearchTool** | `ToolSearchTool/` | Deferred tool discovery—searches ranked tool schemas on-demand to avoid context overflow |

### Workflow & Control

| Tool | Files | Purpose |
|------|-------|---------|
| **ExitTool** | `ExitTool/` | Gracefully terminates the session, flushing message history and releasing resources |
| **IgnoreTool** | `IgnoreTool/` | Manages `.claudeignore` patterns, both file-level and global (~/.claudeignore) |
| **ClearTool** | `ClearTool/` | Clears conversation history with optional scope (current conversation, all, artifacts) |
| **BuiltinTool** | `BuiltinTool/` | Exposes core AI capabilities (Notebook, Composer, Artifact) as opt-in tools |

### Infrastructure & Utilities

| Module | Files | Purpose |
|--------|-------|---------|
| **agentToolUtils** | `agentToolUtils.ts`, `loadAgentsDir.ts` | Dynamic tool resolution, agent definition loading from filesystem, `agentFile` metadata parsing |
| **MCP** | `MCP/` | MCP server lifecycle management (stdio, HTTP/SSE), tool and resource exposure, permission integration |
| **ToolCallCacher** | `ToolCallCacher.ts` | Batches and deduplicates consecutive tool calls of the same type for display efficiency |
| **shared** | `shared/parse.ts`, `shared/format.ts` | Common parsing (input schemas, task types) and formatting (diffs, file paths, ANSI, ANSI-safe truncation) utilities |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Tool Registration                                │
│                   (tools/index.ts → orchestrator)                           │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
        ┌────────────────────────┼────────────────────────────────────────┐
        │                        │                                        │
        ▼                        ▼                                        ▼
┌───────────────────┐  ┌─────────────────────────┐  ┌──────────────────────┐
│  Orchestrator     │  │   MCP/                  │  │  agentToolUtils.ts   │
│  (spawns tools)   │  │   (MCP server mgmt)     │  │  (dynamic resolution)│
└───────────────────┘  └────────────┬────────────┘  └──────────┬───────────┘
                                    │                            │
                                    │                            │
                    ┌───────────────┼────────────────┐           │
                    ▼               ▼                ▼           ▼
            ┌─────────────┐  ┌──────────────┐  ┌────────────┐ ┌──────────┐
            │  AgentTool  │  │ WebFetchTool │  │WebSearchTool│ │ToolSearch │
            │  (sub-agent │  │ (URL fetch   │  │(API search │  │Tool (deferred│
            │   spawning)│  │  + LLM       │  │ streaming) │  │ discovery)│
            │             │  │  extraction) │  │             │  │          │
            │ +Built-in/  │  └──────────────┘  └─────────────┘ └──────────┘
            │ (explore,   │                                                     │
            │  plan, ...) │         ┌──────────────────────────────────────────┐
            └─────────────┘         │          BashTool/                        │
                                    │ (permissions, security, PTY exec, UI)     │
                                    └────────────┬─────────────────────────────┘
                                                 │
                                    ┌────────────┴────────────────┐
                                    ▼                             ▼
                            ┌──────────────┐              ┌────────────────┐
                            │ ReadTool/   │              │   GlobTool/    │
                            │ (file reads)│              │ (file discovery)│
                            │ GrepTool/   │              └────────────────┘
                            │ (search)    │
                            └──────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│                          Shared Utilities                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ shared/format│  │ shared/parse │  │ ToolCallCache│  │    MCP/      │    │
│  │ (ANSI, diffs)│  │ (schemas)    │  │ (dedup)      │  │ (servers)    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
└────────────────────────────────────────────────────────────────────────────┘
```

**Key dependency chains:**

- **AgentTool → MCP**: When an agent definition declares `mcpServers`, the `MCP/` module initializes those servers and exposes their tools. Servers can be `'dynamic'` (ephemeral, per-agent) or named (memoized, shared).
- **WebFetchTool → Anthropic API**: Uses a secondary model call to extract relevant information from fetched content, not just raw HTML.
- **ToolSearchTool → All tools**: Acts as a meta-tool—its index of tool schemas is populated from all other tool definitions to enable on-demand discovery.
- **agentToolUtils → built-in agents**: `loadAgentsDir()` recursively discovers agent definitions (`.md` files with YAML frontmatter) from the filesystem.
- **ReadTool/GlobTool/GrepTool → shared/ignore**: All respect `.claudeignore` patterns and share the `matchIgnore` utility for consistent filtering.

## Key Takeaways

### Design Patterns

| Pattern | Where Applied |
|---------|---------------|
| **Schema-first** | Every tool defines Zod input/output schemas; validation is mandatory before execution |
| **Deferred loading** | Heavy dependencies (`turndown` in WebFetchTool, tree-sitter in BashTool) are dynamically imported on first use |
| **Permission gating** | BashTool, WebFetchTool, AgentTool all implement permission checks before execution |
| **Streaming UI** | AgentTool, WebSearchTool, BashTool use async generators to stream progress in real-time |
| **Memoization** | MCP servers (by name), tool description caches, glob result caches all use in-memory memoization |
| **LRU caching** | BashTool image cache (50MB), ReadTool file cache (100MB) implement LRU eviction |

### Security Model

The tools implement defense-in-depth across multiple layers:

1. **Input validation** — Zod schemas enforce structure; URL length/content limits in WebFetchTool; path normalization everywhere
2. **Command injection prevention** — BashTool uses tree-sitter AST parsing and pattern matching; WebFetchTool validates redirects to same-origin only
3. **Domain allowlisting** — WebFetchTool maintains an 80+ domain preapproved list; BashTool has a configurable hostname blocklist checked via Anthropic API
4. **Permission prompts** — BashTool, WebFetchTool, and AgentTool all defer to user confirmation for sensitive operations
5. **Path constraints** — BashTool restricts `cd` targets and glob roots to project directories unless `cdToNonProjectDirs` is enabled
6. **Ignore files** — ReadTool, GlobTool, GrepTool all respect `.claudeignore` patterns, preventing unintended file exposure

### Tool Categories by Capability

```
┌────────────────────────────────────────────────────────────────┐
│                      Read-only (no file writes)                │
│  ReadTool · GlobTool · GrepTool · WebSearchTool · WebFetchTool  │
│  AgentTool (read-only variants) · BriefTool · ClearTool        │
├────────────────────────────────────────────────────────────────┤
│                      Write-capable                             │
│  BashTool · AgentTool (generalPurpose) · IgnoreTool            │
├────────────────────────────────────────────────────────────────┤
│                      Workflow control                          │
│  ExitPlanModeTool · ExitTool · AgentTool (spawning)           │
│  AskUserQuestionTool · ToolSearchTool (discovery)              │
└────────────────────────────────────────────────────────────────┘
```

### Performance Considerations

- **BashTool PTY overhead**: PTY allocation adds ~20ms startup cost; used unconditionally for ANSI color support and job control
- **GlobTool caching**: Results cached in-memory with `mtime`-based invalidation; unlimited cache size but bounded by glob call frequency
- **ReadTool cache**: 100MB LRU cache with `mtime`/`size` invalidation; `ReadMultipleTool` issues parallel `readFile` calls
- **MCP server reuse**: Named MCP servers are memoized and shared across all tools; dynamic servers are isolated per-agent and cleaned up after use
- **Tree-sitter initialization**: The parser is lazily loaded and reused across BashTool invocations to amortize startup cost

### Cross-cutting Concerns

- **Platform support**: BashTool uses `crossSpawn` for cross-platform subprocess handling; path separators normalized throughout
- **CLI/TTY detection**: Multiple tools check `isInNonInteractiveEnvironment()` to suppress permission prompts in CI/non-TTY contexts
- **Resource cleanup**: BashTool PTY subprocess, MCP servers, and WebFetchTool caches all implement explicit cleanup in `finally` blocks or via `session.end()` hooks
- **Concurrency safety**: ReadTool, GlobTool, GrepTool, WebSearchTool are safe for concurrent use; BashTool runs one command at a time with configurable parallelism
