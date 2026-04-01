# Summary of `constants/`

## Purpose of `constants/`

The `constants/` directory serves as the **single source of truth** for all immutable configuration values used throughout the Claude Code CLI. It centralizes API limits, beta feature flags, security policies, error tracking, tool access control lists, and structured message markup—ensuring consistency and preventing duplication across the codebase.

## Contents Overview

| File | Domain | Key Role |
|------|--------|----------|
| `apiLimits.ts` | API Constraints | Anthropic API limits for images (5 MB), PDFs (100 pages), media items (100/request) |
| `betas.ts` | Feature Flags | 18+ beta headers enabling experimental features (context, tool search, fast mode, etc.) |
| `common.ts` | Utilities | Date formatting helpers with session-scoped memoization for cache stability |
| `cyberRiskInstruction.ts` | Security Policy | Governance rules for acceptable security assistance vs. harmful activities |
| `errorIds.ts` | Diagnostics | Sequential error IDs for production tracing |
| `errorTypes.ts` | Error Handling | Structured error classification with typed subclasses |
| `modelCatalog.ts` | AI Models | Model registry with configs, capabilities, and context windows |
| `systemPrompts.ts` | Caching | System prompt section resolution with cache-break controls |
| `toolLimits.ts` | Tool Constraints | Size limits for tool results (50K chars default, 200K per message aggregate) |
| `tools.ts` | Access Control | Tool allowlists/blocklists per agent context (async, in-process, coordinator) |
| `turnCompletionVerbs.ts` | UI/UX | Past tense verbs for dynamic turn completion messages |
| `xml.ts` | Message Markup | XML tag constants for structured metadata in messages |

## How Files Relate to Each Other

```
apiLimits.ts ──────────────► Consumed by validation logic
                             (client-side pre-check before API requests)

betas.ts ──────────────────► Consumed by API request handlers
                             (HTTP headers to enable experimental features)

toolLimits.ts ─────────────► Consumed by systemPrompts.ts
                             (influences section computation and cache strategies)

tools.ts ──────────────────► Imports ~30 tool constants from ../tools/*/constants.js
                             (aggregates access control across all tool domains)

systemPrompts.ts ──────────► Uses toolLimits.ts via getPerMessageBudgetLimit()
                             (controls when to truncate/persist tool results)
                             Also resets betaHeaderLatches on /clear

xml.ts ─────────────────────► Consumed by message parsers and renderers
                             (identifies terminal output, task notifications, etc.)
```

**Cross-cutting concerns:**
- `betas.ts` and `toolLimits.ts` both influence API request behavior
- `systemPrompts.ts` manages caching while respecting limits from `toolLimits.ts`
- `tools.ts` defines access boundaries consumed by agent orchestration logic
- `constants/betas.ts` conditionally gates beta headers based on `USER_TYPE` environment variable and feature flags from `bun:bundle`

## Key Takeaways

1. **Dependency-free by design**: Several files (`apiLimits.ts`, `cyberRiskInstruction.ts`, `turnCompletionVerbs.ts`) explicitly avoid external dependencies to prevent circular imports and enable safe consumption across all layers.

2. **Bundle optimization**: Multiple files use TypeScript `as const` assertions and named exports to support **dead code elimination**—build tools can strip unused constants from production bundles.

3. **Dual limit layers**: The codebase distinguishes between:
   - **Client-side limits** (resize images to 2000px, truncate tool summaries to 50 chars)
   - **API-enforced limits** (100 pages per PDF, 100 media items per request)

4. **Feature flag integration**: `betas.ts` and `tools.ts` use `feature()` from `bun:bundle` to gate experimental capabilities, allowing gradual rollout without code changes.

5. **Cache-aware design**: `systemPrompts.ts` and `common.ts` implement memoization with explicit cache-breaking semantics—consumers must document reasons when bypassing the cache.

6. **Error traceability**: `errorIds.ts` maintains sequential IDs with strict increment rules, enabling precise production error attribution via log correlation.

7. **Governance separation**: `cyberRiskInstruction.ts` contains policy logic owned by the Safeguards team, kept distinct from executable code to emphasize that changes require explicit review.
