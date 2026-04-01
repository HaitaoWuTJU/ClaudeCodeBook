# Summary of `utils/bash/`

# `utils/bash/` — Cohesive Directory Summary

## Purpose

The `utils/bash/` directory is a comprehensive **bash security analysis and command execution toolkit** used by Claude CLI to safely parse, validate, and execute shell commands. Its primary goals are:

1. **Fail-closed security parsing** — Parse bash commands and reject anything that can't be statically verified, requiring user confirmation for ambiguous cases
2. **Shell environment portability** — Capture user shell configuration (aliases, functions) and replay it for command execution
3. **Cross-platform compatibility** — Normalize paths, redirections, and shell syntax across Linux/macOS/Windows (Git Bash)
4. **Embedded tool integration** — Bundle tools (ripgrep, bfs, ugrep) with fallback to system equivalents

---

## Contents Overview

| File/Directory | Role | Type |
|-----------------|------|------|
| `parser.ts` | **Entry point** — loads tree-sitter WASM, exports `parseCommandRaw()` | Core parser |
| `ast.ts` | **Security validation** — extracts argv/env/redirects, validates for dangerous patterns | Security layer |
| `bashParser.ts` | **Fallback parser** — pure TypeScript recursive-descent parser (no WASM dependency) | Fallback parser |
| `prefix.ts` | **Partial command handling** — analyzes incomplete/prefix commands for autocomplete | UX/Completion |
| `quote.ts` | **Shell quoting** — converts args to shell-safe quoted strings, handles heredocs | Utilities |
| `toPosix.ts` | **Path normalization** — converts Windows paths to POSIX for shell compatibility | Cross-platform |
| `treeSitterAnalysis.ts` | **AST utilities** — quote context, compound structure, dangerous pattern extraction | Analysis helpers |
| `ShellSnapshot.ts` | **Environment capture** — snapshots user shell config (aliases, functions) to file | Environment |
| `registry.ts` | **Spec schema** — defines `CommandSpec` interface for command specifications | Schema |
| `specs/` | **Command registry** — declarative specs for `alias`, `nohup`, `pyright`, `sleep`, `srun`, `time`, `timeout` | Commands |

---

## Architecture & Data Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              INPUT: Bash command string                       │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  TREE-SITTER PATH (primary)                    FALLBACK PATH (WASM unavailable) │
│  ───────────────────────────────                ──────────────────────────────
│  parser.ts                                     bashParser.ts
│  ├─► loadParser() → Parser                     ├─► makeLexer() → Tokenizer
│  │   (lazy WASM load, abort controller)         │    (context-sensitive lexing)
│  └─► parseCommandRaw() → RootNode               └─► parseProgram() → TsNode
│       (tree-sitter AST)                             (pure TS AST)                        │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  ast.ts — Security Validation Layer                                           │
│  ─────────────────────────────────────                                       │
│  1. Recursive walk over AST (TreeSitterNode or TsNode)                       │
│  2. Extract: argv[], envVars[], redirects[]                                  │
│  3. Validate against danger patterns:                                        │
│     • Command substitution ($(), backticks) → too-complex                   │
│     • Process substitution (<(), >()) → too-complex                          │
│     • Array subscript injection → blocked                                    │
│     • Brace expansion obfuscation → detected                                 │
│     • /proc/environ access → blocked                                         │
│     • jq system() / --from-file → blocked                                    │
│     • Eval-like builtins (eval, exec, source) → blocked                      │
│     • Zsh bypass hooks (~[, =cmd) → blocked                                  │
│  4. Security post-checks:                                                    │
│     • Wrapper stripping (sudo, nice, env, stdbuf)                            │
│     • Empty/placeholder command names                                        │
│     • Shell keyword mis-parsing                                              │
│     • Control character injection                                           │
│  5. Result: SimpleCommand[] | { kind: 'too-complex', reason }                │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  Security Validation Result                                                  │
│  ┌─────────────────┐  ┌──────────────────────┐  ┌─────────────────────────┐ │
│  │ kind: 'simple'  │  │ kind: 'too-complex'   │  │ kind: 'parse-unavailable'│ │
│  │ commands: [...] │  │ reason: string       │  │                         │ │
│  │                 │  │ nodeType?: string    │  │                         │ │
│  └─────────────────┘  └──────────────────────┘  └─────────────────────────┘ │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │ quote.ts    │      │ toPosix.ts  │      │ ShellSnapshot│
   │ (exec prep) │      │ (path fix)  │      │ (env setup) │
   └─────────────┘      └─────────────┘      └─────────────┘
          │                    │                    │
          ▼                    ▼                    ▼
   quote() for args     convert paths         Snapshot file
   heredoc handling      WIN → POSIX           (~/.claude/...)
   NUL → /dev/null      pathsep fix
```

---

## Cross-Cutting Concerns

### Security Philosophy: **Fail-Closed**

```
Unrecognized node type → too-complex → require user permission
```

Any bash feature that can't be statically verified is rejected, not guessed. This includes:
- Command substitutions (`$(...)`, <code>\`...\`</code>)
- Process substitutions (`<(...)`, `>(...)`)
- Parameter expansions with metacharacters (`$VAR` with `*?[]`)
- Compound statements (loops, conditionals)
- Heredocs with variable expansion

### Parser Abstraction

Two parser implementations share the same validation layer:

```
tree-sitter-bash WASM          Pure TypeScript fallback
    (parser.ts)                    (bashParser.ts)
         │                              │
         ▼                              ▼
   TreeSitterNode[]                 TsNode[]
         │                              │
         └──────────────┬───────────────┘
                        ▼
                  ast.ts validation
```

The `treeSitterAnalysis.ts` module provides unified utilities for both AST types.

### Quote Context System

The codebase tracks three quote context modes for commands:

| Context | Single Quotes | Double Quotes | Content |
|---------|--------------|----------------|---------|
| `withDoubleQuotes` | **Removed** | Keep delimiters, keep content | Preserves interpolatable content |
| `fullyUnquoted` | **Removed** | **Removed** | All literal — safe for injection points |
| `unquotedKeepQuoteChars` | Keep `'`, remove content | Keep `"`, remove content | `$` kept for ansi_c (`$'...'`) |

### Cross-Platform Path Handling

| Platform | Example | POSIX Conversion |
|----------|---------|------------------|
| Windows | `C:\Users\...` | `/c/Users/...` |
| Cygwin | `/usr/bin` | unchanged |
| Git Bash | `/c/Program Files` | `/c/Program Files` |
| Redirect | `2>nul` | `2>/dev/null` |

---

## Key Design Patterns

### 1. Async Initialization with Caching
```typescript
// parser.ts: Lazy WASM load, cached promise
let parserPromise: Promise<Parser> | null = null;
export function parseCommandRaw(...): RootNode {
  ensureParserInitialized(); // no-op if already loaded
  return parser.parse(source);
}
```

### 2. Defense-in-Depth Validation
Multiple overlapping checks for the same vulnerability:
- **Array subscript injection**: Detected at parse level (SUBSCRIPT_EVAL_FLAGS, BARE_SUBSCRIPT_NAME_BUILTINS) AND via regex (BARE_VAR_UNSAFE_RE)
- **Brace expansion**: Quote-aware scanner + regex pattern matching

### 3. Shell-Aware Quote Scanning
```typescript
// ast.ts: Quote-aware brace expansion detection
maskBracesInQuotedContexts(cmd: string): string
// Prevents: echo '{$(whoami),b}' → whoami executed via brace expansion
// Blocked because { is inside quotes → treated as literal
```

### 4. Environment Snapshot for Portability
```
User's .zshrc/.bashrc
        │
        ▼ (sourced in isolated subshell)
┌─────────────────────────────────────┐
│ Capture:                            │
│ • Functions (typeset -f / declare)  │
│ • Shell options (setopt / shopt)    │
│ • Aliases (unalias conflicts)       │
│ • PATH                              │
└─────────────────────────────────────┘
        │
        ▼
~/.claude/shell-snapshots/snapshot-*.sh
        │
        ▼ (sourced before command execution)
User's environment reproduced
```

### 5. Node Budget & Timeout Safety
```typescript
// bashParser.ts: OOM and DoS protection
const MAX_NODES = 50_000;    // abort if tree exceeds this
const PARSE_TIMEOUT_MS = 50; // wall-clock timeout
```

---

## Dependency Graph

```
┌─────────────────────────────────────────────────────────────────┐
│                        Consumers                                 │
│   (CLI runner, help generator, autocomplete)                    │
└────────────────────────────┬────────────────────────────────────┘
                             │ import
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      specs/index.ts                              │
│              (aggregates all CommandSpec objects)               │
└────────────────────────────┬────────────────────────────────────┘
                             │ import
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     registry.ts                                  │
│              (defines CommandSpec type)                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      ShellSnapshot.ts                            │
│   imports: cleanupRegistry, envUtils, embeddedTools, ripgrep    │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│    ast.ts         │    │  bashParser.ts   │    │ treeSitterAnalysis│
│   imports:        │    │  imports:         │    │  imports:         │
│   ├─ parser.ts    │◄───┤   (no external)   │───►│   (no external)   │
│   └─ bashParser.js│    │                   │    │                   │
│        (types)    │    └──────────────────┘    └──────────────────┘
└──────────────────┬──────────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                       parser.ts                                  │
│   imports: (WASM loading, no internal deps)                     │
│   exports: parseCommandRaw(), PARSE_ABORTED                      │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│    quote.ts       │    │    toPosix.ts     │    │    prefix.ts     │
│   imports:        │    │   imports:        │    │   imports:        │
│   (no internal)   │    │   (no internal)   │    │   └─ parser.ts   │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

---

## Notable Implementation Details

### tree-sitter vs Pure-TS Parser Selection
| Feature | tree-sitter | Pure TypeScript |
|---------|-------------|-----------------|
| Grammar accuracy | Full bash grammar | Hand-written approximation |
| Dependencies | WASM binary (~300KB) | None |
| Error recovery | Built-in | Limited |
| Performance | Fast (WASM) | Moderate |
| Used when | WASM available | WASM unavailable / denied |

### The ARGV0 Dispatch Pattern
Embedded tools (ripgrep, bfs, ugrep) use bun's internal dispatch:
```bash
# bun checks argv[0] at startup and dispatches to embedded tool
ARGV0=rg /path/to/bun --help   # runs ripgrep instead
```

Shell wrappers handle this:
```bash
# Zsh: ARGV0 env var prefix
(ARGV0=rg; rg --help)

# Bash: exec -a (fails in subshells on Windows)
exec -a rg /path/to/bun --help
```

### Unicode/UTF-8 Handling
All byte offset calculations use UTF-8 byte positions, not JS string indices:
```typescript
// Fast ASCII path: byte == char index
const isAscii = srcBytes === src.length;

// Slow path: build lookup table for UTF-8 positions
```

### Heredoc Quoting Strategy
```typescript
// quote.ts: Safe heredoc handling
<<'EOF'          → keep as-is (literal content)
<<"EOF"          → strip quotes, keep EOF delimiter
<<\EOF           → strip backslash escape, keep EOF
<<$VAR           → keep (variable expansion in delimiter)

Heredoc content:
  <<'EOF'         → 'literal content'
  <<EOF          → quoted to prevent history expansion
```

### Security Validation Hierarchy
```
Parse Tree
    │
    ├── DANGEROUS_TYPES encountered? → too-complex
    │       (command_substitution, subshell, expansion, etc.)
    │
    ├── Unrecognized node types? → too-complex
    │
    ├── BARE_VAR with metacharacters? → too-complex
    │       ($VAR containing *, ?, [, ])
    │
    ├── Contains control chars? → too-complex
    │       (\x00-\x08, \x0B-\x1F, \x7F)
    │
    ├── Security post-checks (regardless of parse result):
    │   ├── sudo/nice/env/stdbuf wrapper → strip and re-validate
    │   ├── Empty command name? → reject
    │   ├── Placeholder command? → reject
    │   ├── Array subscript in dangerous position? → reject
    │   ├── jq system() or --from-file? → reject
    │   ├── /proc/*/environ access? → reject
    │   ├── eval-like builtins? → reject
    │   └── zsh bypass hooks? → reject
    │
    └── All checks pass → simple (safe to execute)
```

---

## Summary

The `utils/bash/` directory implements a **layered bash security analysis system**:

1. **Parsing layer** (`parser.ts`, `bashParser.ts`) provides AST generation with safety budgets
2. **Validation layer** (`ast.ts`) enforces fail-closed security policies through comprehensive pattern matching
3. **Analysis layer** (`treeSitterAnalysis.ts`, `prefix.ts`) extracts semantic information for autocomplete and policy decisions
4. **Execution prep layer** (`quote.ts`, `toPosix.ts`) normalizes commands for safe shell execution
5. **Environment layer** (`ShellSnapshot.ts`) ensures user shell context is preserved
6. **Command registry** (`specs/`) provides typed CLI schemas for documented commands

The system prioritizes **security over flexibility**—any ambiguous or complex bash feature requires explicit user permission rather than risking execution of unintended commands.
