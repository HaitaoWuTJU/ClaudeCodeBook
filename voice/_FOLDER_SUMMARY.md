# Summary of `voice/`

## Purpose of `voice/`

Provides feature-gating and authentication utilities for voice mode functionality in the Claude desktop application, enabling or disabling voice interactions based on GrowthBook feature flags and OAuth token validation.

## Contents Overview

The `voice/` subdirectory contains TypeScript utilities for managing voice mode availability:

| File | Role |
|------|------|
| `voice-mode.ts` | Main feature-gating logic combining GrowthBook flags and OAuth auth checks |

### Key Functions in `voice-mode.ts`

1. **`isVoiceGrowthBookEnabled()`** — Reads the `tengu_amber_quartz_disabled` GrowthBook flag. Returns `true` when the flag is absent/stale (safe default), and `false` only when explicitly disabled.

2. **`hasVoiceAuth()`** — Validates that the user has a valid Anthropic OAuth token stored in the macOS Keychain via `getClaudeAIOAuthTokens()`.

3. **`isVoiceModeEnabled()`** — Logical AND of both checks; returns `true` only when the user has valid OAuth **and** the GrowthBook feature flag allows voice mode.

## How Files Relate to Each Other

```
voice-mode.ts
    │
    ├──► growthbook.js (analytics service)
    │         └── getFeatureValue_CACHED_MAY_BE_STALE() — reads feature flag
    │
    └──► auth.js
              ├── getClaudeAIOAuthTokens() — reads Keychain via macOS security
              └── isAnthropicAuthEnabled() — checks auth provider config
```

- `voice-mode.ts` acts as a **gatekeeper**, aggregating decisions from the analytics and auth subsystems
- No circular dependencies exist between files
- External consumers (commands, config UI) import from `voice-mode.ts` without needing to know about GrowthBook or Keychain internals

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Gating layers** | Two distinct checks: visibility (GrowthBook flag only) vs. usage (auth + flag). Voice requires both. |
| **OAuth requirement** | Voice mode is unavailable to API keys, Bedrock, Vertex, or Foundry — only Anthropic OAuth tokens work. |
| **Safe defaults** | Stale/missing GrowthBook cache defaults to `false` (disabled), which means voice is **enabled** by default for new users. |
| **Performance** | Auth token lookup spawns a `security` process (~20–50ms) but is memoized for ~1 hour. |
| **Build compatibility** | Uses positive ternary pattern (`condition ? !value : false`) to avoid string literal inlining by external build tools. |
| **No writes** | This module only reads from GrowthBook cache and macOS Keychain; no side effects. |
