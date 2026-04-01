# Summary of `schemas/`

## Purpose of `schemas/`

The `schemas/` directory contains Zod validation schemas that define the structure and constraints for hook configurations used throughout the application. This directory was created to break circular dependency cycles that arose when `src/utils/settings/types.ts` and `plugins/schemas.ts` both tried to import from each other.

## Contents Overview

**`schemas/hooks.ts`** is the sole file in this directory. It provides:

1. **`IfConditionSchema`** - A reusable conditional filter using permission rule syntax (e.g., `"Bash(git *)"`) to control hook execution conditions

2. **`buildHookSchemas()` factory** - Generates four distinct hook type schemas:
   - `BashCommandHookSchema` → Shell command hooks (`type: "command"`)
   - `PromptHookSchema` → LLM prompt evaluation hooks (`type: "prompt"`)
   - `HttpHookSchema` → HTTP webhook hooks (`type: "http"`)
   - `AgentHookSchema` → Agentic verification hooks (`type: "agent"`)

3. **Union and composite schemas**:
   - `HookCommandSchema` - Discriminated union of all four hook types
   - `HookMatcherSchema` - Matcher configuration containing multiple hooks
   - `HooksSchema` - Top-level hooks configuration (partial record of hook events)

4. **Type exports** - Inferred TypeScript types via `z.infer<>` for use throughout the codebase

## How Files Relate to Each Other

```
HOOK_EVENTS (agentSdkTypes.js)
         │
         ▼
┌─────────────────────────┐
│   schemas/hooks.ts      │
│                         │
│  ┌───────────────────┐  │
│  │ IfConditionSchema │  │◄── Shared conditional logic
│  └─────────┬─────────┘  │
│            │            │
│  ┌─────────▼─────────┐  │
│  │ buildHookSchemas()│  │◄── Factory for 4 hook types
│  └─────────┬─────────┘  │
│            │            │
│  ┌─────────▼─────────┐  │
│  │ HookCommandSchema │  │◄── Discriminated union
│  └─────────┬─────────┘  │
│            │            │
│  ┌─────────▼─────────┐  │
│  │   HooksSchema     │  │◄── Top-level config
│  └───────────────────┘  │
└─────────────────────────┘
         │
         ▼
   TypeScript types exported for use elsewhere
```

The file consumes `HOOK_EVENTS` from `src/entrypoints/agentSdkTypes.js` and `SHELL_TYPES` from `../utils/shell/shellProvider.js`, then exports validated types for consumers like `plugins/schemas.ts`.

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Dependency strategy** | Uses `lazySchema()` wrapper to defer schema evaluation and avoid circular imports |
| **HTTP header interpolation** | Supports `$VAR_NAME` / `${VAR_NAME}` syntax in HTTP hook headers, but only for variables in `allowedEnvVars` |
| **AgentHookSchema warning** | A known bug (gh-24920) prevents using `.transform()` on this schema—JSON round-tripping silently drops user prompts |
| **Async execution** | Hooks support `async: true` for non-blocking execution and `asyncRewake: true` for waking the model on exit code 2 |
| **Type inference** | Exports both schemas (for validation) and inferred types (for TypeScript usage) |
| **Discriminated unions** | `HookCommandSchema` uses the `type` field to discriminate between hook implementations |
