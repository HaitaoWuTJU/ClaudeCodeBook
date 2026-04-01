# Claude Code Book
[English](README.md) | [简体中文](README_zh.md)

We have automatically generated the Claude Code Book: [book.pdf](book.pdf)"

## Overview
> Claude Code: An AI-powered CLI agent for software development, built by Anthropic.

Claude Code is a command-line interface tool that brings Claude's reasoning capabilities directly into the development workflow. It provides an interactive REPL with AI assistance, a comprehensive tool system for file operations and command execution, Vim-style modal editing, and deep integration with development workflows (Git, shell commands, editors).

The project is written in TypeScript with Bun as the runtime and build tool. It targets multiple platforms (macOS, Linux, Windows) and supports multiple authentication methods including OAuth, API keys, and cloud provider integrations (AWS Bedrock, Google Vertex AI).

## Repository Structure

```
claude-code/
├── src/                          # Main application source code
│   ├── index.ts                  # Application entry point
│   ├── commands.ts               # Slash command registry and loading
│   ├── context.ts                # AI context preparation (system/user prompts)
│   ├── cost-tracker.ts           # API usage tracking and cost reporting
│   ├── tools.ts                  # Tool registry and MCP integration
│   │
│   ├── commands/                  # Slash command implementations (~40 commands)
│   │   ├── add.ts                 # Add/edit code with AI
│   │   ├── autofixPr.ts          # Auto-fix pull requests
│   │   ├── bash.ts               # Shell command execution
│   │   ├── config.ts             # Configuration management
│   │   ├── diff.ts               # Git diff viewer
│   │   ├── git.ts                # Git operations
│   │   └── ...
│   │
│   ├── tools/                     # Tool implementations for AI agent
│   │   ├── BashTool.ts
│   │   ├── FileReadTool.ts
│   │   ├── WebSearchTool.ts
│   │   ├── AgentTool.ts
│   │   └── ...
│   │
│   ├── utils/                     # Shared utilities (~40 subdirectories)
│   │   ├── bash/                  # Shell parsing, security, execution
│   │   │   ├── exec.ts           # Command execution engine
│   │   │   ├── parser.ts         # Shell parser utilities
│   │   │   ├── security.ts       # Command security validation
│   │   │   └── shellSnapshot.ts  # Shell state capture
│   │   ├── git/                  # Git operations
│   │   ├── settings/             # Configuration persistence
│   │   ├── analytics/            # Session analytics and BigQuery export
│   │   ├── telemetry/            # OpenTelemetry tracing and metrics
│   │   ├── teleport/             # Remote session API client
│   │   ├── vim/                  # Vim emulation layer
│   │   ├── todo/                 # Todo data types
│   │   ├── ultraplan/            # Ultra-plan workflow
│   │   └── ...
│   │
│   ├── bootstrap/                 # Application initialization
│   │   └── state.ts              # Global STATE singleton
│   │
│   ├── bridge/                    # Remote control infrastructure
│   │   ├── bridgeApi.ts          # Environments REST API client
│   │   ├── bridgeConfig.ts       # Configuration with OAuth/keychain
│   │   ├── bridgeDebug.ts        # Fault injection for testing
│   │   └── workSecret.ts        # Work secret decoding and session routing
│   │
│   ├── buddy/                     # Companion mascot system
│   │   ├── companion.js          # Creature generation (seeded PRNG)
│   │   ├── sprites.ts           # ASCII art rendering
│   │   └── CompanionSprite.tsx  # React sprite component
│   │
│   ├── assistant/                # Teleport Agent SDK
│   │   ├── entrypoints/         # SDK entry points and types
│   │   └── utils/               # API client with OAuth support
│   │
│   ├── voice/                     # Voice mode feature gating
│   │   └── voice-mode.ts        # GrowthBook + OAuth auth checks
│   │
│   └── vim/                       # Vim emulation
│       ├── transitions.ts        # State machine (NORMAL/INSERT modes)
│       ├── motions.ts           # Cursor movement functions
│       ├── operators.ts         # Edit operations (delete, yank, etc.)
│       ├── textObjects.ts       # Text object boundaries
│       └── types.ts             # TypeScript interfaces and constants
│
├── src/commands.ts               # Command registry entry
├── src/tools.ts                  # Tool registry entry
├── src/context.ts                # Context preparation entry
├── src/cost-tracker.ts           # Cost tracking entry
├── src/index.ts                   # Main entry point
│
├── package.json                  # Node.js dependencies
├── bunfig.toml                   # Bun build configuration
├── tsconfig.json                 # TypeScript configuration
└── ...
```

## Getting Started

### Prerequisites

- **Bun** ≥ 1.0 (install via [bun.sh](https://bun.sh))
- **Node.js** ≥ 18 (for fallback)
- **macOS** / **Linux** / **Windows** (WSL supported)

### Installation

```bash
# Clone the repository
git clone https://github.com/anthropics/claude-code.git
cd claude-code

# Install dependencies with Bun
bun install

# Build the project
bun run build

# Link for development
bun link
```

### Development Setup

```bash
# Run in development mode with hot reload
bun run dev

# Run tests
bun test

# Run with debug output
DEBUG=* bun run start
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `ANTHROPIC_API_KEY` | Anthropic API key | Required for API key auth |
| `DEBUG` | Enable debug logging | - |
| `BUN_ENV` | Environment (`development`, `production`) | `production` |
| `OTEL_LOG_USER_PROMPTS` | Include user prompts in telemetry | `false` |
| `BETA_SESSION_TRACING` | Enable beta tracing attributes | `false` |

## Usage

### Basic Commands

```bash
# Start an interactive session
claude

# Start with a specific task
claude "Explain this codebase"

# Start with auto-approve mode (for automation)
claude --approve-all "Review and fix bugs"

# Resume a previous session
claude --resume <session-id>
```

### Slash Commands

Available in the interactive REPL:

| Command | Description |
|---------|-------------|
| `/add [description]` | Add or edit code based on description |
| `/bash <command>` | Execute a shell command |
| `/diff [file]` | Show uncommitted changes |
| `/git <subcommand>` | Run a git command |
| `/config [key] [value]` | View or modify configuration |
| `/clear` | Clear the conversation |
| `/help` | Show help information |
| `/model <name>` | Switch the active model |
| `/compact` | Compact context window |
| `/session` | Show current session info |

### Configuration

```bash
# View current config
claude config list

# Set a config value
claude config set model claude-opus-4-5

# Set an environment variable
claude config set --env API_KEY your_key
```

## Key Modules & Scripts

### Core Entry Points

| File | Purpose |
|------|---------|
| `src/index.ts` | Main entry point; orchestrates initialization, session management, and the REPL loop |
| `src/commands.ts` | Central registry for all slash commands with feature-gated loading |
| `src/tools.ts` | Tool registry assembling built-in tools, MCP tools, and permission-filtered tools |
| `src/context.ts` | Prepares system and user context for AI conversations |
| `src/cost-tracker.ts` | Tracks API usage, costs, and generates usage reports |

### State Management

| Module | Purpose |
|--------|---------|
| `src/bootstrap/state.ts` | Global STATE singleton with typed accessors for all runtime state |
| `src/utils/settings/settings.ts` | Project-level configuration persistence |

### Security & Bash Execution

| Module | Purpose |
|--------|---------|
| `src/utils/bash/security.ts` | Fail-closed security model; rejects unverifiable shell constructs |
| `src/utils/bash/exec.ts` | Command execution with streaming, timeout, and cancellation |
| `src/utils/bash/shellSnapshot.ts` | Captures shell state for context injection |

### Telemetry & Analytics

| Module | Purpose |
|--------|---------|
| `src/utils/telemetry/index.ts` | Public API for OpenTelemetry tracing and metrics |
| `src/utils/telemetry/sessionTracing.ts` | Span management for interactions, LLM calls, and tools |
| `src/utils/telemetry/bigqueryExporter.ts` | Exports metrics to BigQuery |
| `src/utils/analytics/sessionAnalytics.ts` | Session-level analytics with BigQuery export |

### Vim Emulation

| Module | Purpose |
|--------|---------|
| `src/vim/transitions.ts` | State machine coordinating NORMAL/INSERT modes |
| `src/vim/motions.ts` | Pure functions for cursor movement |
| `src/vim/operators.ts` | Pure functions for edit operations |
| `src/vim/textObjects.ts` | Functions for text object boundaries |
| `src/vim/types.ts` | TypeScript interfaces and state type definitions |

### Remote & Cloud Features

| Module | Purpose |
|--------|---------|
| `src/bridge/bridgeApi.ts` | Environments REST API client for remote control |
| `src/bridge/bridgeConfig.ts` | Configuration with OAuth and keychain integration |
| `src/assistant/` | Teleport Agent SDK for session management |
| `src/utils/teleport/api.ts` | Sessions API client for remote environments |

## Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI Interface                           │
│   User Input → Slash Commands → REPL → Vim Mode                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    Command Registry (commands.ts)               │
│   Resolves slash commands, checks feature flags, validates perms  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     Tool Registry (tools.ts)                    │
│   Built-in tools + MCP tools + permission-filtered tools        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    Tool Implementations                          │
│   BashTool, FileReadTool, WebSearchTool, AgentTool, ...         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    AI Model (Claude API)                         │
│   Context prepared by context.ts                                │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    Cost Tracking (cost-tracker.ts)              │
│                    Telemetry (telemetry/)                        │
│                    Analytics (analytics/)                       │
└─────────────────────────────────────────────────────────────────┘
```

### Authentication Model

Claude Code supports multiple authentication methods:

1. **OAuth** — User login via `claude auth login` (stored in macOS Keychain)
2. **API Key** — Set via `ANTHROPIC_API_KEY` environment variable
3. **Cloud Providers** — AWS Bedrock, Google Vertex AI, AWS IAM
4. **Trusted Device** — Device tokens for remote control (CCR v2)

### Feature Gating

Features are gated via:
- **GrowthBook** (`growthbook.js`) — Remote feature flags for A/B testing
- **Bundle constants** (`bun:bundle`) — Dead-code elimination for disabled features
- **Platform checks** — macOS-only features (Keychain, speech synthesis)

## Data Pipeline

### Context Preparation Flow

```
1. User input received
       │
2. context.ts → getUserContext()
       │        ├── Reads .claude/CLAUDE.md
       │        ├── Reads .claude/instructions.md
       │        └── Memoized per session
       │
3. context.ts → getSystemContext()
       │        ├── Git status (branch, recent commits)
       │        ├── Current directory info
       │        └── Cache breaker injection
       │
4. Combined into system prompt injection
```

### Telemetry Flow

```
Tool execution / LLM call
       │
Span created by sessionTracing.ts
       │
Events logged by events.ts (logOTelEvent)
       │
Metrics accumulated by OpenTelemetry SDK
       │
bigqueryExporter.ts → POST to BigQuery API
       │
(Optionally) Perfetto trace file written
```

### Session Analytics Flow

```
Session starts
       │
Analytics initialized with session ID
       │
Per-tool/per-command metrics accumulated
       │
Session ends
       │
bigqueryExporter.ts → POST to BigQuery
       │
STATE.sessionAnalytics cleared
```

## Dependencies

### Runtime Dependencies

| Package | Purpose |
|---------|---------|
| `typescript` | TypeScript language support |
| `@anthropic-ai/sdk` | Anthropic API client |
| `@modelcontextprotocol/sdk` | MCP protocol implementation |
| `@anthropic-ai/growthbook` | Feature flag client |
| `@google-cloud/bigquery` | BigQuery analytics export |
| `@aspect-build/rules_ts` | TypeScript build rules |
| `@aspect/registry` | Aspect registry |
| `@opentelemetry/*` | OpenTelemetry SDK |
| `@vueuse/core` | Vue composition utilities |
| `tree-sitter-bash` | Shell parsing |
| `zod` | Schema validation |
| `uuid` | UUID generation |
| `semver` | Semantic version parsing |
| `jose` | JWT handling |
| `ora` | Terminal spinners |
| `chalk` | Terminal colors |
| `glob` | File globbing |
| `diff` | Text diffing |
| `yaml` | YAML parsing |
| `zx` | Shell script utilities |

### Development Dependencies

| Package | Purpose |
|---------|---------|
| `bun` | Runtime and build tool |
| `vitest` | Unit testing framework |
| `@types/node` | Node.js type definitions |
| `ts-node` | TypeScript execution |
| `eslint` | Linting |
| `prettier` | Code formatting |

## License

**Proprietary** — Claude Code is developed and maintained by Anthropic PBC. Usage is subject to Anthropic's Acceptable Use Policy and Terms of Service.

For commercial licensing or enterprise deployments, contact Anthropic at [anthropic.com](https://anthropic.com).