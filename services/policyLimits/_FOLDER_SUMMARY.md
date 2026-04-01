# Summary of `services/policyLimits/`

## Purpose of `policyLimits/`

This directory implements a **Policy Limits Service** that fetches organization-level policy restrictions from the Claude API and uses them to control which CLI features are available. It follows patterns similar to remote managed settings, including fail-open behavior, ETag-based HTTP caching, background polling, and retry logic.

## Contents Overview

| File | Description |
|------|-------------|
| `types.ts` | TypeScript types and Zod schemas for API responses and fetch results |
| `index.ts` | Main service implementation with caching, polling, and policy checking logic |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                      types.ts                                │
│  • PolicyLimitsResponseSchema (Zod validation)               │
│  • PolicyLimitsResponse (inferred TypeScript type)           │
│  • PolicyLimitsFetchResult (fetch result shape)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       index.ts                                │
│                                                               │
│  ┌─────────────────┐    ┌─────────────────┐                  │
│  │  API Responses  │───▶│  Zod Schemas    │ (from types.ts) │
│  └─────────────────┘    └─────────────────┘                  │
│                                                               │
│  Main exports:                                                │
│  • isPolicyAllowed()        ← Core feature gate API          │
│  • loadPolicyLimits()       ← Initialization                 │
│  • refreshPolicyLimits()    ← Auth-aware refresh             │
│  • startBackgroundPolling() ← Hourly refresh                 │
│  • clearPolicyLimitsCache() ← Cleanup                        │
└─────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Eligibility-based**: Only Console API key users or OAuth users with Team/Enterprise subscriptions are checked for policy limits.

2. **Multi-layer caching**:
   - Session cache (in-memory, synchronous access)
   - File cache (`~/.config/claude-code/policy-limits.json`)
   - HTTP cache (ETag via `If-None-Match`)

3. **Fail-open default**: If the API is unavailable and a cache exists, stale data is used. Only HIPAA's essential-traffic mode enforces fail-closed for specific policies.

4. **Background refresh**: Polls the API every hour to catch mid-session policy changes without blocking CLI operations.

5. **Retry with backoff**: Network failures trigger up to 6 retries with exponential backoff, but authentication errors skip retry immediately.
