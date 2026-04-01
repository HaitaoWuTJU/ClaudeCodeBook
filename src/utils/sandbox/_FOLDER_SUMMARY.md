# Summary of `utils/sandbox/`

## Purpose of `sandbox/`

The `sandbox/` directory provides Claude CLI's integration layer for **containerized process isolation** via the external `@anthropic-ai/sandbox-runtime` package. It acts as a bridge between Claude Code's settings system, tool integrations, and the underlying sandbox runtime (bubblewrap on Linux, sandbox-exec on macOS). The directory also exposes a small UI utility for scrubbing sandbox violation metadata from error messages.

## Contents Overview

| File | Role |
|------|------|
| `sandbox-adapter.ts` | Core adapter — converts Claude Code settings → sandbox runtime config, initializes sandbox manager, enforces security protections, manages state |
| `sandbox-ui-utils.ts` | UI utility — removes `<sandbox_violations>` tags from error messages for clean user-facing display |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│  Settings (permissions, sandbox.*, policy, local, etc.)      │
└──────────────────────────┬──────────────────────────────────┘
                           │ read (sandbox-adapter.ts)
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  sandbox-adapter.ts                                            │
│  • convertToSandboxRuntimeConfig() → SandboxRuntimeConfig     │
│  • initialize() → SandboxManager (handles process lifecycle)  │
│  • Security hardening (settings.json, skills, bare git repos) │
│  • Platform checks, dependency verification, state management │
└──────────┬─────────────────────────────────┬────────────────┘
           │ call                            │ call
           ▼                                 ▼
┌──────────────────────┐       ┌─────────────────────────────┐
│ sandbox-runtime      │       │ Error messages / output     │
│ (bubblewrap/socat)   │       │ (e.g. violation exceptions) │
└──────────────────────┘       └──────────────┬──────────────┘
                                              │ sanitize
                                              ▼
                                  ┌─────────────────────────┐
                                  │ sandbox-ui-utils.ts     │
                                  │ removeSandboxViolation  │
                                  │ Tags()                  │
                                  └─────────────────────────┘
```

- **`sandbox-adapter.ts`** owns the full integration pipeline: reading settings, converting them to sandbox runtime format, initializing the sandbox manager, and applying security hardening.
- **`sandbox-ui-utils.ts`** is a standalone utility consumed downstream (e.g., in CLI output formatting) to strip technical violation tags before presenting errors to users.

## Key Takeaways

1. **Adapter Pattern**: `sandbox-adapter.ts` is not the sandbox implementation itself — it wraps `@anthropic-ai/sandbox-runtime`, adding Claude Code-specific logic for settings resolution, path conventions, and security hardening.

2. **Dual Path Semantics**: The adapter must resolve two different path conventions depending on context:
   - Permission rules → relative to settings file (`/path`)
   - `sandbox.filesystem.*` settings → absolute (`/path` = root-relative)

3. **Security-Independent Protections**: Several protections are injected into the sandbox config regardless of user settings to prevent sandbox escape:
   - Settings files (`.claude/settings.json`)
   - Skills directory (`.claude/skills`)
   - Bare git repository contents (`HEAD`, `objects`, `refs`, `hooks`, `config`)

4. **Dynamic Configuration**: The adapter subscribes to settings change events and synchronously refreshes sandbox config on every settings change, ensuring sandbox restrictions stay in sync with policy updates.

5. **Platform-Gated Feature**: Sandbox support requires specific platform + tool combinations (macOS with sandbox-exec, Linux/WSL2 with bubblewrap). The adapter checks availability at runtime and surfaces clear unavailability reasons to users.

6. **Deferred Cleanup**: For transient protections (e.g., bare git repo files planted during sandbox init), the adapter uses a deferred scrub list cleaned up post-command via `cleanupAfterCommand()`.

7. **UI Sanitization is Stateless**: `sandbox-ui-utils.ts` is a pure, stateless function with no dependencies on the rest of the sandbox subsystem — it exists only to clean error output for end users.
