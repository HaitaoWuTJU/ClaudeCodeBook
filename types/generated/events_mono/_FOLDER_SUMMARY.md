# Summary of `types/generated/events_mono/`

## Purpose of `events_mono/`

The `events_mono/` directory is the **typed, protobuf-generated event schema layer** for GrowthBook experiment tracking within the broader `events_mono` monorepo. It provides canonical TypeScript interfaces and serialization utilities that represent the exact data shapes sent to analytics pipelines (Statsig → BigQuery). By isolating these schemas in a monorepo, multiple services can share a single source of truth for experiment event types, eliminating duplication and schema drift.

---

## Contents Overview

| File | Description |
|------|-------------|
| `common/v1/auth.ts` | Shared `PublicApiAuth` type — carries `account_id`, `organization_uuid`, and `account_uuid` for identity attribution across all event types |
| `growthbook/v1/growthbook_events.ts` | `GrowthbookExperimentEvent` type — captures a single A/B test exposure: experiment ID, variation assignment, user context, and timing |

Both files follow the same `ts_proto` output pattern: an interface, a base factory, a full factory, a partial updater, and bidirectional JSON converters.

---

## How Files Relate to Each Other

```
events_mono/
├── common/
│   └── v1/
│       └── auth.ts              ← Shared identity payload (PublicApiAuth)
└── growthbook/
    └── v1/
        └── growthbook_events.ts ← Experiment event (embeds PublicApiAuth)
                                      └── imports PublicApiAuth for partial
                                          deserialization helpers
```

**Import dependency:**
- `growthbook_events.ts` imports `PublicApiAuth` from `../../common/v1/auth.ts` only for its use in `fromPartial()` — the `GrowthbookExperimentEvent` interface does **not** embed `PublicApiAuth` as a field; instead it carries three top-level identity fields (`account_id`, `account_uuid`, `organization_uuid`) that mirror the auth payload structurally.

**Usage chain:**
1. A server receives a GrowthBook experiment assignment from its SDK
2. It builds a `GrowthbookExperimentEvent` using `create()` or `fromPartial()`
3. The event is serialized via `toJSON()` and sent to the internal logging endpoint
4. The Statsig/BigQuery pipeline reads the JSON and maps fields to their SQL columns

---

## Key Takeaways

1. **Auto-generated, not hand-written** — Both files are output of `protoc-gen-ts_proto`. The authoritative source is the `.proto` definition; any change belongs there first.

2. **Two-level versioning strategy**
   - Top-level `events_mono/` tracks the monorepo version
   - Inner `v1/` tracks the schema version, so consumers pin to `growthbook/v1/*` and `common/v1/*` explicitly

3. **Shared `common/` module prevents duplication** — `PublicApiAuth` lives in `common/v1/` and is imported where needed rather than redefined per service, ensuring a single identity schema across all event types.

4. **All fields are optional** — Every field in both interfaces uses `?:`, so callers can construct events incrementally. The `fromPartial()` function is the canonical entry point for partial updates.

5. **JSON is the primary wire format** — `fromJSON()` / `toJSON()` are the main serialization path. Protobuf binary encoding is not used because clients log to an HTTP endpoint expecting JSON payloads.

6. **Long integer handling** — `growthbook_events.ts` imports a `Long` type (not shown in this excerpt), indicating that large numeric IDs (e.g., `experiment_id`) may exceed JavaScript's `Number.MAX_SAFE_INTEGER` and must be handled as strings or `Long` objects to avoid precision loss.
