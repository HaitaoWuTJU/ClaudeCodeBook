# Summary of `services/oauth/`

## Purpose of `oauth/`

Implements a complete OAuth 2.0 authorization code flow with PKCE (Proof Key for Code Exchange) for authenticating users with Claude services. This subsystem supports both **automatic** (browser-based) and **manual** (copy-paste code) authentication flows, handles token refresh, and retrieves user profile information.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | **Orchestrator** — `OAuthService` coordinates the entire flow, from PKCE generation through token exchange to profile fetching |
| `client.ts` | **OAuth Client** — Builds authorization URLs, exchanges codes for tokens, refreshes tokens, fetches profile, manages account info |
| `crypto.ts` | **PKCE Utilities** — Generates code verifiers, code challenges (SHA-256 + Base64URL), and CSRF state strings |
| `auth-code-listener.ts` | **Redirect Capture** — Creates a temporary localhost HTTP server to intercept OAuth callbacks containing the authorization code |
| `getOauthProfile.ts` | **Profile API** — Low-level HTTP callers for fetching user profile using either an API key or an OAuth access token |
| `types.ts` | **TypeScript Interfaces** — Shared type definitions for tokens, profiles, and configuration objects |
| `constants.ts` | **OAuth Configuration** — URLs, client ID, scope constants, and success/error redirect URLs |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OAuthService (index.ts)                            │
│  Entry point. Orchestrates the full flow from PKCE → token exchange →       │
│  profile fetch, coordinating both automatic and manual authentication.       │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐
│    crypto.ts    │  │auth-code-listener │  │      client.ts       │
│                  │  │      .ts          │  │                     │
│ PKCE generation: │  │                   │  │ • buildAuthUrl()     │
│ • code_verifier  │  │ Captures OAuth    │  │ • exchangeCode()     │
│ • code_challenge │  │ redirect on       │  │ • refreshToken()     │
│ • state param    │  │ localhost,        │  │ • fetchProfile()     │
│                  │  │ extracts code     │  │ • storeAccountInfo() │
└──────────────────┘  │ & state.          │  │                     │
                      │                   │  │ Also uses:           │
                      │ Uses:             │  │ • crypto.ts          │
                      │ • client.ts       │  │ • getOauthProfile.ts │
                      │   (scope check)   │  │ • constants.ts       │
                      └───────────────────┘  └──────────┬──────────┘
                                                        │
                                                        ▼
                                            ┌─────────────────────┐
                                            │  getOauthProfile.ts  │
                                            │                     │
                                            │ Fetches profile via: │
                                            │ • API key + UUID    │
                                            │ • OAuth access token │
                                            └─────────────────────┘
```

**Flow Sequence:**

1. **`index.ts`** generates PKCE via `crypto.ts` (code verifier + challenge) and a state parameter
2. **`index.ts`** builds authorization URLs via `client.ts.buildAuthUrl()`
3. **`index.ts`** starts `auth-code-listener.ts` on a localhost port to capture the redirect
4. **`index.ts`** opens browser (or returns URLs for SDK mode via `authURLHandler`)
5. User authenticates in browser; OAuth provider redirects to `localhost:PORT/callback?code=...&state=...`
6. **`auth-code-listener.ts`** extracts the code and state, validates CSRF, and signals readiness
7. **`index.ts`** exchanges the code + verifier for tokens via `client.ts.exchangeCodeForTokens()`
8. **`client.ts`** optionally refreshes tokens and fetches profile via `getOauthProfile.ts`
9. **`client.ts`** stores account info (uuid, email, orgUuid, billing type) to global config
10. **`auth-code-listener.ts`** sends a 302 redirect to a success/error URL based on granted scopes

## Key Takeaways

- **PKCE Security**: All authorization requests use Proof Key for Code Exchange (RFC 7636) with SHA-256 challenges, protecting against authorization code interception.
- **Dual Flow Support**: The system gracefully handles both automatic (browser + redirect) and manual (user paste) flows, with the automatic flow serving as the default and manual as a fallback.
- **CSRF Protection**: State parameter validation in `auth-code-listener.ts` ensures the callback originates from a request the service initiated.
- **Profile Caching**: `client.ts.refreshOAuthToken()` skips redundant `/api/oauth/profile` calls when both global config and secure storage already contain the necessary data, reducing API load by an estimated ~7M requests/day.
- **Scope-Based Routing**: After authentication, granted scopes determine the success redirect URL—Claude AI scopes go to `CLAUDEAI_SUCCESS_URL`, while Console/workspace scopes use `CONSOLE_SUCCESS_URL`.
- **SDK Integration**: The `skipBrowserOpen` flag in `index.ts` allows the caller (e.g., the SDK control protocol) to receive both manual and automatic URLs and handle display itself.
- **Environment Variable Fallback**: For SDK callers, `populateOAuthAccountInfoIfNeeded()` in `client.ts` respects pre-set `CLAUDE_CODE_*` environment variables to avoid network calls and prevent telemetry race conditions.
