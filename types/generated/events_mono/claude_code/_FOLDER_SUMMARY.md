# Summary of `types/generated/events_mono/claude_code/`

## Purpose of `claude_code/`

The `claude_code/` directory contains the **protobuf-derived TypeScript interfaces and serialization logic** for Claude Code's internal telemetry events. These schemas are the canonical data shapes used throughout the codebase to log structured events to Statsig (and ultimately BigQuery) for product analytics.

## Contents Overview

| File | Role |
|------|------|
| `claude_code_internal_event.ts` | **Primary schema definition** — defines the top-level `ClaudeCodeInternalEvent` interface along with all nested types, plus their `create`, `fromJSON`, `toJSON`, and `fromPartial` factory/serialization functions |
| `index.ts` | Re-exports all public symbols from the schema file, acting as the package entry point |

## How Files Relate to Each Other

```
claude_code_internal_event.ts
├── EnvironmentMetadata
│   └── GitHubActionsMetadata (embedded)
├── SlackContext
└── ClaudeCodeInternalEvent (main event type)
    ├── public_api_auth: PublicApiAuth (imported from ../../common/v1/auth.js)
    ├── environment_metadata: EnvironmentMetadata
    ├── slack_context: SlackContext
    └── 30+ scalar fields (timestamps, session_id, model, event_type, etc.)

index.ts
└── Re-exports everything from claude_code_internal_event.ts
```

The schema forms a **strict hierarchy**: the top-level `ClaudeCodeInternalEvent` embeds `EnvironmentMetadata` (which itself embeds `GitHubActionsMetadata`) and optionally `SlackContext`, with each layer adding domain-specific context for analytics attribution.

## Key Takeaways

1. **Generated, not hand-written** — This directory is the output of `protoc-gen-ts_proto`; edits belong in the `.proto` source, not here.

2. **Monolithic schema file** — All interfaces and their serializers live in a single `claude_code_internal_event.ts` file (~1600 lines), which is typical of `ts_proto` output. The `index.ts` is just a thin re-export barrel.

3. **Tight Statsig coupling** — The `event_name` field drives downstream filtering in BigQuery against `proj-product-data-nhme.raw_statsig_internal_tools.events`, so enum values (`tengu_binary_feedback`, `tengu_api_success`, `cis_*`) must stay in sync with pipeline queries.

4. **Optional-only fields** — Every field is optional (`T | undefined`), so consumers must always guard against `undefined`.

5. **JSON serialization is the primary wire format** — Both `toJSON` and `fromJSON` are the main interchange path; the protobuf wire format is not used directly since this is a client-side SDK logging to an HTTP endpoint.
