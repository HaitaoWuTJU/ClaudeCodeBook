# Summary of `tools/SyntheticOutputTool/`

## Purpose of `SyntheticOutputTool/`

This directory provides a mechanism for returning **validated, structured output (JSON)** at the end of an agent response in non-interactive contexts (SDK/CLI workflows). Rather than presenting results through an interactive UI, the tool accepts any input, validates it against a caller-supplied JSON Schema, and returns the structured data as part of the agent's final output.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Primary entry point — exports `createSyntheticOutputTool()`, `isSyntheticOutputToolEnabled()`, and all types. Contains the entire tool definition, factory functions, validation logic, and caching mechanism. |

## How Files Relate to Each Other

`index.ts` is self-contained; there are no sibling files in this directory. It depends on the following external utilities:

1. **`ajv`** — Compiles JSON Schema into a fast validator function
2. **`zod`** — Defines lazy input/output type schemas for TypeScript inference
3. **`../../Tool.js`** — Provides `Tool` base class, `ToolDef`, and `ToolInputJSONSchema` interfaces
4. **`../../utils/errors.js`** — Supplies `TelemetrySafeError_I_VERIFIED_THIS_NOT_CODE_OR_FILEPATHS` for validation error reporting
5. **`../../utils/lazySchema.js`** — Defers Zod schema evaluation to avoid circular dependencies
6. **`../../utils/slowOperations.js`** — Provides `jsonStringify` for safe UI rendering of large payloads

## Key Takeaways

- **Design Pattern**: This is a **factory-based tool** where each call to `createSyntheticOutputTool(schema)` produces a distinct tool instance bound to a specific JSON Schema. Multiple calls with the same schema object return a cached instance.
- **Caching Strategy**: A `WeakMap<object, CreateResult>` keyed by the schema object itself ensures that repeated invocations in a single session skip Ajv compilation entirely (~96% overhead reduction after first call).
- **Schema Validation**: The tool first validates the provided schema with `Ajv.validateSchema()` (fail-fast on malformed schemas), then compiles it with `Ajv.compile()`. On each `call()`, the compiled validator runs against the actual input.
- **Non-Interactive Only**: `isSyntheticOutputToolEnabled()` gates tool creation behind the `isNonInteractiveSession` flag, ensuring this tool never appears in interactive chat UIs.
- **Error Handling**: Validation failures throw a `TelemetrySafeError` containing Ajv's formatted error messages (instance path + message per failure).
- **Output Structure**: Successful calls return `{ data: "<max 100k chars>", structured_output: <input> }`, allowing downstream consumers to parse and use the structured result programmatically.
