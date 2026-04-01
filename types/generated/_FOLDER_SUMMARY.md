# Summary of `types/generated/`

## Purpose of `generated/google/`

The `generated/google/` directory contains TypeScript type definitions and utility functions for **Google's Well-Known Types (WKT)** as defined in the protobuf specification. These are standard, language-agnostic data types that protobuf compilers (like `ts_proto`) translate into idiomatic TypeScript, providing the foundation for cross-language compatibility across the entire monorepo's event pipeline.

The `timestamp.ts` file specifically implements the `google.protobuf.Timestamp` type, which represents a point in time with nanosecond precision. All timestamp fields in every event schema (`GrowthbookExperimentEvent`, `PublicApiAuth`, etc.) ultimately resolve to this type, making it the **canonical temporal representation** for the entire analytics system.

---

## Contents Overview

| File | Description |
|------|-------------|
| `timestamp.ts` | Implements `google.protobuf.Timestamp` — carries elapsed time as `seconds` (since Unix epoch) and `nanos` (fractional offset). Includes JSON bidirectional converters, partial updaters, and a base factory. |

The `timestamp.ts` module exposes several types to callers:

| Symbol | Role |
|--------|------|
| `Timestamp` | Primary protobuf message interface with `fromJSON`, `toJSON`, `create`, `fromPartial` methods |
| `TimestampAmino` | Amino (Cosmos SDK–compatible) variant with identical structure but separate serialization path |
| `TimestampSDK` | SDK variant for CosmJS/cometbft integration contexts |
| `Long` | Imported utility for 64-bit integer handling (since JS `number` lacks precision above `Number.MAX_SAFE_INTEGER`) |

---

## How Files Relate to Each Other

```
events_mono/
├── growthbook/
│   └── v1/
│       └── growthbook_events.ts   ──► imports Timestamp from
│                                       ../../generated/google/
│                                           └───► timestamp.ts
│
└── common/
    └── v1/
        └── auth.ts                ──► also uses Timestamp indirectly
                                        via ts_proto generated code
```

**Data flow for any timestamp field:**

```
Application date (Date object)
         │
         ▼
fromJson("2024-01-15T10:30:00.000Z")
         │
         ▼
{ seconds: 1705315800, nanos: 0 }   ←─ Timestamp instance
         │
         ▼
toJson()
         │
         ▼
"2024-01-15T10:30:00.000Z"         ←─ JSON wire format
```

The `Timestamp` type is **never directly imported by consumers**; instead, `ts_proto` generates wrapper code that uses it internally. When a field like `timestamp` in `GrowthbookExperimentEvent` is serialized, the generated `toJSON()` delegates to `Timestamp.toJSON()` under the hood.

---

## Key Takeaways

1. **The root of all temporal fields** — Every timestamp field in every event schema resolves to this one file. Changing its behavior (e.g., timezone handling, serialization format) propagates to all event consumers immediately.

2. **64-bit second handling** — The `seconds` field uses `Long` to avoid JavaScript's floating-point precision loss for timestamps beyond year 2262 (when `Number` would overflow). Consumers should treat `seconds` as opaque and always use `toJSON()` / `fromJSON()` rather than accessing raw fields.

3. **Three type variants for compatibility** — `Timestamp`, `TimestampAmino`, and `TimestampSDK` exist to support different serialization conventions. The standard `Timestamp` is correct for the Statsig/BigQuery pipeline; the others are retained for CosmJS-related integrations.

4. **Zero-value is the default** — `createBaseTimestamp()` initializes to `{ seconds: 0, nanos: 0 }`, which corresponds to the Unix epoch (1970-01-01T00:00:00Z). Generated code uses this as the zero/empty sentinel.

5. **Immutable-ish design** — `create()` returns a frozen object; `fromPartial()` merges provided fields with defaults. This pattern prevents accidental mutation of shared event instances in concurrent contexts.

6. **Auto-generated, never edited manually** — This file is `ts_proto` output. Any structural change must be made in the `.proto` source and regenerated. Manual edits are overwritten on the next build.
