# Summary of `tools/PowerShellTool/`

## Purpose of `tools/PowerShellTool/`

This directory implements a **PowerShell execution tool** for a terminal-based CLI application (built with Ink/React). It provides:

1. **Secure command execution** with a read-only allowlist validation system
2. **Constrained Language Mode (CLM)** type checking to block dangerous .NET types
3. **Destructive command detection** with user-facing warning messages
4. **Exit code interpretation** so tools like `robocopy` and `grep` don't produce false CI failures
5. **Terminal UI rendering** for commands, progress, results, and errors

The tool is designed to safely run PowerShell commands while protecting against:
- Destructive operations (file deletion, system shutdown)
- Data exfiltration (variable expansion, environment variable leakage)
- Dangerous .NET types (WMI, ADSI, reflection, P/Invoke)
- Parser differentials (e.g., `--git-dir` in malicious paths)

---

## Contents Overview

| File | Purpose |
|------|---------|
| **readOnlyValidation.ts** | Core security module—validates cmdlets against `CMDLET_ALLOWLIST`, checks flags/safe flags, detects argument value leakage, dispatches to external command validators |
| **clmTypes.ts** | CLM type allowlist (~100 types) + utilities (`isClmAllowedType`, `normalizeTypeName`) for AST type checking |
| **destructiveCommandWarning.ts** | Pattern-matching detector for destructive operations (git reset/push, rm -rf, Format-Volume, etc.) returning warning strings |
| **commandSemantics.ts** | Exit code interpreter—distinguishes success/failure semantics per command (e.g., `grep 1` = no match, `robocopy 1-7` = success) |
| **UI.tsx** | React/Ink UI renderer—displays commands, progress, stdout/stderr, errors, timeouts, background tasks |
| **commonParameters.ts** | Shared constants for PowerShell common parameters (`-ErrorAction`, `-Verbose`, etc.)—breaks circular import between validation modules |
| **toolName.ts** | Shared `POWERSHELL_TOOL_NAME` constant—breaks circular import in `prompt.ts` |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                        prompt.ts / Tool Invoker                                        │
│                  (calls POWERSHELL_TOOL_NAME from toolName.ts)                        │
└─────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                         UI.tsx                                   │
│  (renders command display, progress, results, errors to terminal) │
└─────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    readOnlyValidation.ts                         │
│  ┌──────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ CMDLET_ALLOW │  │ commonParams│  │ external validators:    │  │
│  │ LIST        │  │ (imported)  │  │ isGitSafe, isGhSafe,    │  │
│  │              │  │             │  │ isDockerSafe, isDotnet  │  │
│  │ + argLeaks   │  │             │  │ (use EXTERNAL_READONLY  │  │
│  │   Value()    │  │             │  │  from shellUtils)       │  │
│  └──────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
           │                    │
           ▼                    ▼
┌──────────────────┐  ┌──────────────────────────────────────────┐
│   clmTypes.ts     │  │       destructiveCommandWarning.ts       │
│  isClmAllowedType │  │  getDestructiveCommandWarning()         │
│  normalizeType   │  │  (returns warning strings for prompts)   │
│  CLM_ALLOWED_TYPES│  │                                          │
└──────────────────┘  └──────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    commandSemantics.ts                           │
│  interpretCommandResult()                                        │
│  (converts exit codes to {isError, message} for UI/status)       │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow (End-to-End)

1. **User invokes PowerShell tool** → `prompt.ts` receives the request
2. **Parse command** → `parser.ts` produces `ParsedPowerShellCommand` AST
3. **Validate command** → `readOnlyValidation.ts` checks:
   - Cmdlet name in `CMDLET_ALLOWLIST`
   - Flags against `safeFlags` (uses `COMMON_PARAMETERS` from `commonParameters.ts`)
   - Arguments for `$variable` leakage via `argLeaksValue()`
   - External commands via `isGitSafe`/`isGhSafe`/`isDockerSafe`/`isDotnetSafe`
4. **CLM type check** (if applicable) → `clmTypes.ts` validates AST type nodes
5. **Destructive check** → `destructiveCommandWarning.ts` returns warning string
6. **Execute command** → External process runs
7. **Interpret exit code** → `commandSemantics.ts` converts exit code to `{isError, message}`
8. **Render UI** → `UI.tsx` displays command, progress, output, status

---

## Key Takeaways

### Security Architecture

- **Layered validation**: Allowlist-first approach with multiple independent checks (cmdlet, flags, arguments, external commands)
- **Prototype pollution prevention**: `CMDLET_ALLOWLIST` uses `Object.create(null)`
- **AST-based detection**: Uses `elementTypes` from parsed AST for parameter detection (catches Unicode dash differentials)
- **Environment-based restrictions**: `gh` commands restricted to `'ant'` user type only

### Notable Removed/Hardened Items

| Item | Reason |
|------|--------|
| `Select-Xml` | XXE vulnerability via `DOCTYPE SYSTEM` references |
| `Test-Json -Schema` | `$ref` can fetch external schemas |
| `Get-Command` / `Get-Help` | Module autoload bypasses validation |
| `adsi`, `adsisearcher` | LDAP network binds |
| `wmi`, `wmiclass`, `wmisearcher`, `cimsession` | Remote WMI execution |
| `DirectoryEntry` (FQ), `ManagementObject` (FQ) | Same hazards in FQ form |
| Arguments containing `$` | Variable expansion can exfiltrate secrets |
| `--attr-source` in git | Parser differential—treated as subcommand but consumed as pathspec |

### Design Patterns

- **Circular dependency breaking**: `commonParameters.ts` and `toolName.ts` exist solely to break import cycles
- **Conservative defaults**: Unknown cmdlets/flags are rejected; unknown exit codes are treated as errors
- **Semantic exit code handling**: Distinguishes "expected non-zero" (`grep 1`, `robocopy 1-7`) from actual failures

### External Integrations

| Tool | Validator | Notes |
|------|-----------|-------|
| `git` | `isGitSafe()` | Global `--config-env` blocked; variable args rejected |
| `gh` | `isGhSafe()` | `'ant'` users only; variable args rejected |
| `docker` | `isDockerSafe()` | Whitelist of read-only subcommands |
| `dotnet` | `isDotnetSafe()` | Read-only flags only (`--version`, `--list-*`) |
