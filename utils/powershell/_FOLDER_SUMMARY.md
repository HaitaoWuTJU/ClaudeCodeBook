# Summary of `utils/powershell/`

## Purpose of `powershell/`

The `powershell/` subdirectory implements PowerShell-specific security analysis, command parsing, and permission-dialog prefix extraction for Claude Code's shell integration. It provides the infrastructure to safely gate dangerous commands, analyze command structure, and generate permission dialog suggestions—mirroring the bash equivalent but tailored to PowerShell's AST, alias system, and cmdlet conventions.

## Contents Overview

| File | Role |
|------|------|
| **`parser.ts`** | Core: spawns a PowerShell process to parse commands via `System.Management.Automation.Language.Parser::ParseInput()`, returning a typed AST (`ParsedPowerShellCommand`). Derives security flags (`hasScriptBlocks`, `hasSubExpressions`, etc.), extracts commands/variables/redirections, and provides utility helpers (`hasCommandNamed`, `commandHasArg`, etc.). Defines `COMMON_ALIASES` (70+ alias → cmdlet mappings). |
| **`dangerousCmdlets.ts`** | Security constants: aggregates all cmdlet sets that execute arbitrary code (`FILEPATH_EXECUTION_CMDLETS`, `DANGEROUS_SCRIPT_BLOCK_CMDLETS`, `SHELLS_AND_SPAWNERS`, etc.) into a single `NEVER_SUGGEST` set, consumed by both the security validator and the UI suggestion gate. |
| **`staticPrefix.ts`** | UI layer: extracts static command prefixes for the "don't ask again for: ___" permission dialog field. Uses `parser.ts` for AST analysis, `dangerousCmdlets.ts` for blocklists, and delegates external commands to the shared fig-spec walker (`buildPrefix`). Collapses compound commands via word-aligned LCP. |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Security / UI Architecture                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  dangerousCmdlets.ts                                         │   │
│  │  • NEVER_SUGGEST = union of all dangerous cmdlet sets        │   │
│  │  • aliasesOf() → resolves aliases from parser.ts             │   │
│  └────────────────────────┬─────────────────────────────────────┘   │
│                           │ exported to:                            │
│           ┌───────────────┴───────────────┐                        │
│           ▼                               ▼                        │
│  ┌─────────────────────┐     ┌─────────────────────────────┐        │
│  │  powershellSecurity │     │  staticPrefix.ts            │        │
│  │  (security guard)   │     │  (permission dialog UI)     │        │
│  └──────────┬──────────┘     └──────────────┬──────────────┘        │
│             │                                │                      │
│             └───────────┬────────────────────┘                      │
│                         ▼                                           │
│             ┌─────────────────────────────┐                          │
│             │  parser.ts                 │                          │
│             │  • parsePowerShellCommand() │                          │
│             │  • COMMON_ALIASES (exported)│                          │
│             │  • SecurityFlags derivation │                          │
│             │  • Utility helpers          │                          │
│             └─────────────────────────────┘                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Dependency chain:**

1. **`parser.ts`** is the foundational module—no other file can function without it. It defines `COMMON_ALIASES` and provides the parsing infrastructure used by both security and UI layers.

2. **`dangerousCmdlets.ts`** imports `COMMON_ALIASES` from `parser.ts` to resolve aliases in `aliasesOf()`. It does **not** import `parsePowerShellCommand`; it only needs the static alias map.

3. **`staticPrefix.ts`** imports:
   - `COMMON_ALIASES` indirectly (via `hasCommandNamed` from parser.ts)
   - `parsePowerShellCommand` directly for AST parsing
   - `NEVER_SUGGEST` from `dangerousCmdlets.ts` to block dangerous cmdlets from prefix suggestions

4. **`powershellSecurity.ts`** (in parent `permissions/`) consumes:
   - `NEVER_SUGGEST` from `dangerousCmdlets.ts` for the allowlist guard
   - `parsePowerShellCommand` from `parser.ts` for AST-based command analysis

**Synchronization point:** `dangerousCmdlets.ts` is the single source of truth for both the security validator (`powershellSecurity.ts`) and the UI suggestion gate (`staticPrefix.ts`). Without this centralization, drift between security enforcement and user-facing suggestions would cause either false negatives (dangerous commands suggested) or false positives (safe commands blocked).

## Key Takeaways

### Security Architecture

- **Defense in depth**: Dangerous cmdlet detection happens at two layers—AST-based validation in `powershellSecurity.ts` (deep analysis of arguments, script blocks, redirections) and prefix-blocklist matching in `staticPrefix.ts` (surface-level prevention of over-broad permission grants).
- **`NEVER_SUGGEST` is the bridge**: By importing the same constant set that the security validator uses, the UI never suggests prefixes that would bypass security (e.g., `Invoke-Expression:*`).
- **`ARG_GATED_CMDLETS` narrow the allowlist**: Some cmdlets (e.g., `Get-Content`) have both safe and dangerous usages. They are not in `NEVER_SUGGEST`; instead, they have `additionalCommandIsDangerousCallback` validators in `powershellSecurity.ts` that gate specific argument patterns.

### Parser Design

- **Native AST over regex**: Using `System.Management.Automation.Language.Parser::ParseInput()` provides accurate tokenization, handles edge cases (backtick-escaped parameters, colon-bound arguments, splatting), and avoids regex brittleness.
- **Timeout isolation**: Parse operations run in a child `pwsh` process with a configurable 5s timeout, preventing parser bugs from hanging Claude Code.
- **Security flag derivation**: The `deriveSecurityFlags()` pass walks the AST once to classify script blocks, subexpressions, expandable strings, and member invocations—enabling security rules to reject commands that dynamically construct code.

### Prefix Extraction

- **Fig-spec reuse**: External commands (non-cmdlets) are delegated to `buildPrefix()` from `../shell/specPrefix.js`, the same machinery used for bash. This ensures consistent prefix generation across shells.
- **Compound command handling**: Commands like `npm run build && git status` are split, prefixes are extracted per-segment, and shared roots are collapsed via `wordAlignedLCP` to produce minimal, precise permission prefixes.
- **Alias transparency**: The parser resolves aliases bidirectionally (`sc` → `Set-Content` and vice versa), ensuring that user commands using aliases get canonical cmdlet prefixes (or `NEVER_SUGGEST` blocklist matching).

### Known Historical Fixes (Documented in `parser.ts`)

The PowerShell parser had several blind spots that were discovered through security testing:

- **`param()` block evaluation**: Script-level `param()` default-value expressions and `[ValidateScript({...})]` attributes run at invocation time but exist outside the pipeline AST walker.
- **Colon-bound arguments**: `CommandParameterAst.Argument` children require special handling for `:` syntax (e.g., `-InputObject:$var`).
- **`using` statements**: `using module` and `using assembly` load external code and execute top-level bodies—outside the pipeline scope.
- **Alias collision on Windows**: `sc` (→ `sc.exe`), `sort` (→ `sort.exe`), `curl`/`wget` are intentionally **not** mapped in `COMMON_ALIASES` to prevent auto-allowing native executables that shadow PowerShell cmdlets.
