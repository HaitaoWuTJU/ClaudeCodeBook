# Summary of `utils/model/`

## Purpose of `utils/model/`

This directory is the central hub for all AI model-related logic in the Claude Code codebase. It manages model selection, resolution, validation, and configuration across multiple API providers (first-party Anthropic, AWS Bedrock, Google Vertex, Microsoft Foundry). The modules handle everything from type-safe model aliases to AWS Bedrock inference profile discovery.

## Contents Overview

| File | Primary Responsibility |
|------|------------------------|
| `agent.ts` | Determines which model subagents should use (inheritance, aliases, Bedrock region prefixes) |
| `aliases.ts` | Defines and validates model alias strings (`sonnet`, `opus`, `haiku`, `best`) |
| `antModels.ts` | Provides Ant-specific model configurations from feature flags |
| `bedrock.ts` | AWS Bedrock client creation, inference profile fetching, region prefix utilities |
| `effort.ts` | Effort level types and validation (off, low, medium, high, auto) |
| `modelStrings.ts` | Maps canonical model IDs to provider-specific strings, applies user overrides |
| `modelSupportOverrides.ts` | Checks capability overrides for 3rd-party models via env vars |
| `providers.ts` | Detects active API provider from environment variables |
| `validateModel.ts` | Validates model availability via allowlists and API test calls |

## How Files Relate to Each Other

```
providers.ts (detection)
     │
     ├─────────────────────────────────────────┐
     │                                         │
     ▼                                         ▼
bedrock.ts ◄─────────────┐            modelStrings.ts
     │                   │                     │
     │                   │                     │
     ▼                   │                     │
agent.ts ◄───────────────┤              aliases.ts
     │                   │                     │
     │                   │                     │
     └───────────────────┘                     │
          (cross-region inheritance)           │
                                              ▼
                                     validateModel.ts
                                              │
                                              ▼
                                       antModels.ts
```

**Dependency hierarchy (top = foundational):**

1. **`providers.ts`** — Base detection layer. All other modules call `getAPIProvider()` to determine behavior.

2. **`aliases.ts`** — Type-safe constants and validation utilities used by nearly every module.

3. **`bedrock.ts`** — Provider-specific logic for AWS Bedrock. Creates clients, fetches inference profiles, handles region prefixes.

4. **`effort.ts`** — Shared type definitions for effort levels, used in agent and model configuration.

5. **`modelStrings.ts`** — Bridges canonical model definitions (`configs.ts`) with runtime string resolution, applies user overrides from `settings.json`.

6. **`modelSupportOverrides.ts`** — Reads env vars to inject capability overrides for third-party providers.

7. **`agent.ts`** — Orchestrates model resolution for subagents. Combines:
   - Provider detection (`providers.ts`)
   - Alias resolution (`aliases.ts`)
   - Bedrock region prefix inheritance (`bedrock.ts`)
   - Model string resolution (`modelStrings.ts`)
   - Capability overrides (`modelSupportOverrides.ts`)

8. **`validateModel.ts`** — Uses `aliases.ts`, `modelStrings.ts`, and makes API calls to validate model availability.

9. **`antModels.ts`** — Highest-level abstraction. Feature-flagged models for internal Ant users, builds on the validation/selection infrastructure.

## Key Takeaways

- **Multi-provider abstraction**: The architecture cleanly separates provider detection (`providers.ts`) from model resolution, allowing Bedrock/Vertex/Foundry-specific logic to be isolated in dedicated modules.

- **Region prefix inheritance**: A distinctive design pattern in `agent.ts` automatically propagates AWS Bedrock cross-region inference prefixes (e.g., `eu.`, `us.`) to subagents, ensuring IAM-scoped permissions remain valid across the agent hierarchy.

- **Override layering**: `modelStrings.ts` implements a three-layer system: built-in defaults → dynamic Bedrock profiles → user `settings.json` overrides. This allows runtime configuration without code changes.

- **Lazy initialization**: Bedrock profile fetching is asynchronous and non-blocking; built-in defaults are returned immediately while profiles load in the background.

- **Memoization throughout**: Both `modelSupportOverrides.ts` and `bedrock.ts` use `memoize` to cache expensive operations (env var parsing, AWS API calls).

- **Strict typing**: `aliases.ts` uses `as const` and type guards to ensure only validated alias strings flow through the system, catching errors at compile time.

- **Ant-specific isolation**: `antModels.ts` gates all functionality behind `USER_TYPE === 'ant'`, keeping experimental models completely separate from production code paths.
