# Summary of `tools/BashTool/`

# BashTool Directory Summary

## Purpose of `BashTool/`

The `BashTool` directory implements a complete shell command execution system for a CLI application (Claude Code). It provides:

- **Permission management** — User-defined rules controlling which commands can execute
- **Security validation** — Protection against command injection, path traversal, and dangerous patterns
- **Command execution** — Integration with a PTY-based shell subprocess
- **Output handling** — Image processing, truncation, formatting, and progress display
- **User interface** — React components for rendering commands and results in a terminal environment

---

## Contents Overview

| File | Size | Purpose |
|------|------|---------|
| `index.ts` | ~21 KB | Main entry point: tool definition, configuration, command execution via PTY |
| `bashPermissions.ts` | ~98 KB | Core permission system: rule matching, security checks, path constraints, suggestions |
| `bashSecurity.ts` | ~25 KB | Deprecated command injection prevention via AST analysis |
| `bashCommandHelpers.ts` | ~12 KB | Helper functions for segmented/compound command permission checking |
| `bashPermissions_DEPRECATED.ts` | ~13 KB | Legacy string-based permission rule matching |
| `UI.tsx` | ~25 KB | React UI components for command display, progress, errors |
| `utils.ts` | ~7 KB | Image processing, output formatting, cwd management |
| `toolName.ts` | <1 KB | Constant export to break circular dependencies |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                          BashTool/index.ts                          │
│  • Defines tool schema (BashTool.inputSchema, BashTool.name)         │
│  • Configures tool capabilities (cdToNonProjectDirs, outputRedirection)│
│  • Orchestrates command execution                                    │
│  • Delegates permission/security to helper modules                   │
└─────────────────────────────────────────────────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐
│ bashPermissions │  │  bashSecurity    │  │      UI.tsx     │
│  (permissions)  │  │  (security)      │  │  (rendering)    │
└─────────────────┘  └──────────────────┘  └─────────────────┘
          │
          ▼
┌─────────────────────────┐  ┌──────────────────────────┐
│   bashCommandHelpers    │  │ bashPermissions_DEPRECATED│
│ (segmented commands)    │  │   (legacy matching)       │
└─────────────────────────┘  └──────────────────────────┘
                                      │
                                      ▼
                               ┌─────────────┐
                               │   utils.ts  │
                               │ (utilities) │
                               └─────────────┘
```

**Data Flow:**

1. **Entry**: `index.ts` receives a command via `BashToolInput`
2. **Permission**: `bashToolHasPermission()` checks rules via `bashPermissions.ts`
3. **Security**: `bashSecurity.ts` validates via tree-sitter AST (command injection)
4. **Segmented Commands**: `bashCommandHelpers.ts` handles piped/compound commands
5. **Execution**: If allowed, runs via PTY subprocess in `index.ts`
6. **Output**: `utils.ts` formats/truncates; `UI.tsx` renders for terminal display

---

## Key Takeaways

### 1. Layered Permission Architecture

The permission system has **three layers**:
- **Bash deny/ask rules** — User-defined patterns (`prefix:exact:*`, `wildcard:*`)
- **Path constraints** — Project boundary validation via `pathInAllowedWorkingPath()`
- **Security checks** — Command injection detection via tree-sitter AST

### 2. Security Features

- **Command injection prevention**: Tree-sitter AST analysis in `bashSecurity.ts` catches malicious patterns
- **Path traversal protection**: `pathInAllowedWorkingPath()` prevents escaping project directories
- **Bare repo protection**: `cd+git` cross-segment detection prevents fsmonitor bypass in bare repositories
- **Sed constraints**: Validation of sed parameters (especially `-i` edits)

### 3. Complex Command Handling

- **Compound commands**: Split by `&&`, `||`, `|`, `;` and each segment checked independently
- **Redirection stripping**: Output redirections removed before security checks to prevent filename injection
- **Multiple cd detection**: Prompts approval for commands changing directory multiple times
- **Fixed-point algorithm**: Iteratively strips safe wrappers (`nice`, `timeout`, env vars) until no new patterns emerge

### 4. Image Output Pipeline

`utils.ts` implements a sophisticated pipeline:
- Detects base64 data URIs in stdout
- Resizes high-DPI images (e.g., matplotlib at `dpi=300`)
- Caps at 20 MB to prevent memory issues
- Falls back to disk file when output truncates

### 5. Tree-Sitter AST Parsing

Commands are parsed into an AST for:
- **Command extraction**: Separate segments from operators/redirections
- **Security analysis**: Identify subshell/command group structures
- **Semantic checking**: Validate command semantics (e.g., `cat` doesn't write to files)

### 6. Tree-Sitter AST Parsing

Commands are parsed into an AST for:
- **Command extraction**: Separate segments from operators/redirections
- **Security analysis**: Identify subshell/command group structures
- **Semantic checking**: Validate command semantics (e.g., `cat` doesn't write to files)

### 7. Legacy Support

Multiple `DEPRECATED` functions exist for backwards compatibility:
- `bashCommandIsSafeAsync_DEPRECATED()` — Old string-based security check
- `bashToolCheckCommandOperatorPermissions_DEPRECATED()` — Legacy compound command handling
- `splitCommand_DEPRECATED()` — Old command splitting
- `bashToolHasPermission_DEPRECATED()` — Full legacy permission flow

### 8. Output Limits

- **Max command display**: 2 lines / 160 characters (truncated)
- **Max tool output**: Via `getMaxOutputLength()` (truncation with marker)
- **Max image size**: 20 MB file reads, ~5 MB after base64 encoding
