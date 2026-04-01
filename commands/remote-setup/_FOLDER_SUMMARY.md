# Summary of `commands/remote-setup/`

## Purpose of `remote-setup/`

This directory implements the **remote setup command** (`remote-setup`) for connecting a user's local GitHub credentials to "Claude on the web" (CCR — Claude Code Remote). It handles the end-to-end flow: verifying local authentication, importing a GitHub token to the CCR backend, creating a default cloud environment, and launching the web interface.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command configuration entry point — gates access via feature flags (`tengu_cobalt_lantern`) and policy limits (`allow_remote_sessions`), lazy-loads the implementation. |
| `api.ts` | Core business logic — token import API, environment creation, OAuth state checks, and a `RedactedGithubToken` wrapper for secure token handling. |
| `remote-setup.tsx` | React/CLI UI — multi-step state machine (`checking` → `confirm` → `uploading`) that orchestrates the user's experience and analytics. |

## How Files Relate to Each Other

```
index.ts (command config)
  │
  ├── isEnabled(): checks feature flag + policy
  │
  └── load(): dynamic import → remote-setup.tsx
                                │
                                ├── Step 1: useEffect → api.isSignedIn() + getGhAuthStatus()
                                │
                                ├── Step 2: User confirms → api.importGithubToken(RedactedGithubToken)
                                │
                                ├── Step 3: api.createDefaultEnvironment() (best-effort)
                                │
                                └── Step 4: openBrowser(api.getCodeWebUrl())
                                                   │
                                                   └── api.ts (shared utilities)
```

1. **`index.ts`** acts as the gatekeeper — it checks whether the user should see this command at all (via GrowthBook feature flags and policy limits). When the command runs, it lazily loads `remote-setup.tsx`.

2. **`remote-setup.tsx`** is the orchestrator. It calls into `api.ts` for all server-side operations and manages the terminal UI state machine. It uses the `RedactedGithubToken` class to safely handle tokens without leaking them to logs.

3. **`api.ts`** contains the pure business logic. It abstracts away HTTP communication (axios), OAuth headers, token validation, and environment provisioning. The `RedactedGithubToken` class ensures that even if the token appears in an error message or JSON output, it is redacted.

## Key Takeaways

- **Security-first token handling**: `RedactedGithubToken` overrides `toString()`, `toJSON()`, and `util.inspect.custom` to prevent raw tokens from ever appearing in logs or error outputs.
- **Feature-gated rollout**: The command is hidden and disabled behind both a GrowthBook feature flag (`tengu_cobalt_lantern`, potentially stale) and a policy check (`allow_remote_sessions`).
- **Beta header for API calls**: Token imports include an `anthropic-beta: ccr-byoc-2025-07-29` header to opt into the BYOC beta endpoint.
- **Resilient UX**: Environment creation is best-effort — failures are silently ignored, and the web app routes users through env-setup on landing.
- **Analytics on every path**: All user actions (cancel, errors, success) are logged with `tengu_remote_setup_started` and `tengu_remote_setup_result` events.
- **State machine pattern**: The UI uses a discriminated union `Step` type to enforce exhaustive handling of each state in the flow.
