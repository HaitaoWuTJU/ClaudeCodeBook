# Summary of `utils/teleport/`

## Purpose of `teleport/`

The `teleport/` directory implements Claude Code's remote session infrastructure for CCR (Claude Code Repository). It provides APIs and utilities for creating, managing, and interacting with remote development environments — enabling users to run Claude Code sessions on cloud infrastructure rather than locally.

## Contents Overview

| File | Primary Responsibility |
|------|----------------------|
| `api.ts` | Core Sessions API client — fetches sessions, sends events, updates titles, handles retry logic |
| `environments.ts` | Environment management — fetches available environments, creates default cloud environments |
| `environmentSelection.ts` | Resolution logic — determines which environment is selected and traces its settings source |
| `gitBundle.ts` | Repository packaging — creates Git bundles for seeding remote sessions with local context |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                     External Consumers                               │
│  • CLI commands (init, session commands)                             │
│  • CCR environment-runner processes                                  │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  environmentSelection.ts                                             │
│  ├── Fetches: environments from environments.ts                      │
│  ├── Reads: settings from ../settings/                               │
│  └── Outputs: EnvironmentSelectionInfo (which env, where configured) │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  environments.ts                                                     │
│  ├── Fetches: environment list via environments.ts (shared module)   │
│  ├── Calls: api.ts → getOAuthHeaders() for auth                     │
│  └── Outputs: EnvironmentResource[]                                 │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  api.ts                                                              │
│  ├── Reads: sessions from Sessions API (/v1/sessions)               │
│  ├── Calls: ../auth.js → getClaudeAIOAuthTokens()                   │
│  ├── Uses: ../detectRepository.js → parseGitHubRepository()         │
│  └── Outputs: CodeSession[], sends events, updates titles           │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  gitBundle.ts                                                        │
│  ├── Reads: git repository state via utils/git.js                   │
│  ├── Calls: api.ts → uploadFile() for transport                     │
│  ├── Uses: utils/execFileNoThrow.js for git commands                │
│  └── Outputs: bundle file to Files API                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Authentication Flow

All API calls require OAuth authentication (no API key fallback):

```
getClaudeAIOAuthTokens() → prepareApiRequest() → getOAuthHeaders()
                                              → org UUID from getOrganizationUUID()
```

### Session Seeding Flow

When a remote session starts with the local repo as context:

```
1. gitBundle.ts: createAndUploadGitBundle()
   └── Creates Git bundle (full history → HEAD → squashed)
       └── Uploads to Files API via uploadFile()

2. api.ts: sendEventToRemoteSession()
   └── Sends bundle file ID + repo context to session
   └── Session runner uses refs/seed/stash for WIP
```

## Key Takeaways

### 1. OAuth-Based Authentication
Every API interaction requires valid OAuth tokens from `getClaudeAIOAuthTokens()` and an organization UUID. The module rejects API key usage and provides clear error messages for expired sessions.

### 2. Retry with Exponential Backoff
Network requests use a 4-retry strategy (2s → 4s → 8s → 16s) for transient failures. Client errors (4xx) fail immediately; server errors (5xx) are retried.

### 3. Progressive Degradation for Git Bundles
Large repositories degrade gracefully: full history → current branch only → single snapshot. The 100 MB default limit is configurable via feature flags.

### 4. Settings Source Tracing
`environmentSelection.ts` can determine not just which environment is selected, but **which settings source** (local config, Claude.ai settings, flags) contains that configuration.

### 5. Beta Feature Flag
All API requests include `'anthropic-beta': 'ccr-byoc-2025-07-29'` indicating this is early-stage BYOC (Bring Your Own Cloud) functionality.
