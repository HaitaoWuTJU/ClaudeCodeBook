# Summary of `utils/shell/`

## Purpose of `utils/shell/`

Provides a unified shell execution abstraction layer for the Claude CLI, supporting both **Bash** and **PowerShell** with security hardening, environment isolation, sandboxing, and Fig spec-based command prefix resolution.

## Contents Overview

| File | Role |
|------|------|
| `shellProvider.ts` | Interface/type definitions — declares `ShellProvider` contract (type, path, exec builder, env overrides) |
| `bashProvider.ts` | Bash implementation — snapshot sourcing, extglob disable, tmux socket isolation, CWD tracking |
| `powershellProvider.ts` | PowerShell implementation — base64 encoding for sandbox, `-ExecutionPolicy`, CWD tracking |
| `powershellDetection.ts` | Linux/Windows detection — `pwsh` vs `powershell`, snap launcher workaround |
| `outputLimits.ts` | Bounded env var — `BASH_MAX_OUTPUT_LENGTH` with 30k–150k bounds |
| `shellToolUtils.ts` | Tool gating — Windows-only PowerShell tool, ant vs external user defaults |
| `resolveDefaultShell.ts` | Settings accessor — reads `settings.defaultShell`, falls back to `'bash'` |
| `specPrefix.ts` | Command prefix extractor — Fig spec parsing for git/npm/kubectl prefix depth |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Shell Provider Architecture                      │
└─────────────────────────────────────────────────────────────────────┘

    shellProvider.ts                    bashProvider.ts
    (Interface & types) ──────────────► (Bash implementation)
         │                                      │
         │                                      ├── reads: ShellSnapshot
         │                                      ├── reads: sessionEnvironment
         │                                      ├── reads: tmuxSocket
         │                                      ├── reads: bashPipeCommand
         │                                      └── writes: cwd temp files
         │
         │                              powershellProvider.ts
         └─────────────────────────────► (PowerShell implementation)
                                           ├── reads: powershellDetection
                                           ├── reads: sessionEnvVars
                                           └── writes: cwd temp files

    shellToolUtils.ts                           powershellDetection.ts
    (PowerShell gating) ─────────────────────► (Executable resolution)
         │                                            │
         │                                            ├── reads: fs/promises
         │                                            ├── reads: platform
         │                                            └── caches result
         │
         │                              outputLimits.ts
         └─────────────────────────────► (Env var bounds)
                   │
                   └────────────────► (Used by shell executor for truncation)

    resolveDefaultShell.ts
    (Settings reader) ──────────────► (Injects default shell into UI/tool routing)

    specPrefix.ts
    (Fig spec parser) ──────────────► (Used by shell providers to build prefixes
                                        for git/npm/kubectl commands)
```

## Key Takeaways

1. **Dual-shell abstraction**: The `ShellProvider` interface is the contract; both `bashProvider.ts` and `powershellProvider.ts` implement it, enabling consistent command building, spawning, and environment management regardless of runtime.

2. **Security hardening**: Bash provider disables `extglob`, rewrites Windows null redirects (`2>nul` → `2>/dev/null`), and wraps sandboxed commands in base64 to prevent injection.

3. **Lazy isolation**: TMUX socket initialization is deferred until first tmux usage (tool invocation or command execution), avoiding unnecessary startup overhead on non-tmux systems.

4. **Platform asymmetry**:
   - PowerShell is **Windows-only** and gated by `USER_TYPE` (ant/external) and env flags.
   - Bash runs everywhere but uses different snapshots and environment sourcing per platform.
   - Linux snap launcher hang is worked around by probing direct install paths.

5. **Configurable limits**: Output truncation is bounded (30k–150k) via `BASH_MAX_OUTPUT_LENGTH`, preventing runaway buffer issues while allowing tuning.

6. **Spec-aware prefixing**: `specPrefix.ts` enables deep command parsing (e.g., `git push origin main` → prefix `git push`) using Fig autocomplete specs, extracted from bash-only origins for reuse in PowerShell and external tool wrappers.
