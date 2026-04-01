# Summary of `utils/background/remote/`

## Purpose of `remote/`

The `remote/` subdirectory implements the **background remote session infrastructure** for the Claude CLI's teleport functionality. It validates prerequisites before remote sessions start and provides a centralized entry point for remote session eligibility checks.

## Contents Overview

| File | Role |
|------|------|
| `preconditions.ts` | Individual precondition validation functions (auth, Git state, GitHub access, environments) |
| `remoteSession.ts` | Orchestrates precondition checks, defines session types, and exposes `checkBackgroundRemoteSessionEligibility()` as the main public API |

## How Files Relate to Each Other

```
remoteSession.ts (orchestrator)
    │
    └── imports & calls preconditions.ts functions:
        ├── checkNeedsClaudeAiLogin()
        ├── checkHasRemoteEnvironment()
        ├── checkHasGitRemote()
        ├── checkGithubAppInstalled()
        └── checkIsGitClean()
```

**`remoteSession.ts`** acts as the facade — it runs all preconditions in parallel and aggregates results into a unified error list. **`preconditions.ts`** contains the granular, single-responsibility checks that can also be imported independently elsewhere in the codebase.

## Key Takeaways

- **Tiered access model**: The system checks three access methods in order: GitHub App installation → token sync (feature-flagged) → no access
- **Fast-fail on policy**: Policy violations short-circuit all other checks, providing immediate feedback
- **Bundle seeding bypass**: A feature flag (`tengu_ccr_bundle_seed_enabled`) and environment variables (`CCR_FORCE_BUNDLE`, `CCR_ENABLE_BUNDLE`) allow bypassing GitHub-specific checks when bundle seeding is active
- **Parallel validation**: Independent preconditions (login, environment, repository) run concurrently via `Promise.all` for performance
- **AbortSignal support**: GitHub API calls accept cancellation signals, enabling graceful timeout handling
- **Debug-friendly**: Extensive logging via `logForDebugging()` helps troubleshoot precondition failures in development
