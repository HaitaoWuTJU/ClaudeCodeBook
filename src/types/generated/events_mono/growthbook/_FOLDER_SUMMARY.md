# Summary of `types/generated/events_mono/growthbook/`

## Summary: `growthbook/` Directory

### Purpose of `growthbook/`

The `growthbook/` directory houses auto-generated TypeScript interfaces and utilities for **GrowthBook experiment event tracking**. It serves as a typed, JSON-serializable bridge between the backend protobuf definitions and TypeScript/JavaScript clients. The module enables tracking user assignments to A/B test variations, capturing the data needed for experiment analysis and analytics pipelines.

---

### Contents Overview

The `v1/growthbook_events.ts` file is the core of this directory:

| Component | Purpose |
|-----------|---------|
| `GrowthbookExperimentEvent` interface | 12-field record of an experiment exposure event |
| `createBaseGrowthbookExperimentEvent()` | Factory for creating default-initialized instances |
| `create()` / `fromPartial()` | Construct instances from full or partial data |
| `fromJSON()` / `toJSON()` | Convert between protobuf objects and JSON |
| Runtime type helpers | `Long`, `Timestamp`, `PublicApiAuth` imports for numeric/string handling |

---

### How Files Relate to Each Other

```
v1/growthbook_events.ts
├── imports ──► ../../common/v1/auth.ts (PublicApiAuth for partial object handling)
└── imports ──► ../../../google/protobuf/timestamp.js (Timestamp for date fields)
```

The file is **self-contained and stateless**. All exported functions operate on the `GrowthbookExperimentEvent` type. The two external dependencies are used only:

1. **`PublicApiAuth`** — Referenced in `fromPartial()` for authentication-aware partial deserialization
2. **`Timestamp`** — Imported for type compatibility when serializing/deserializing date fields

Consumers import the main export directly:

```typescript
import { GrowthbookExperimentEvent } from './growthbook/v1/growthbook_events';
```

---

### Key Takeaways

1. **Auto-Generated Code** — This directory is produced by `protoc-gen-ts_proto`. Manual edits will be overwritten on the next regeneration run.

2. **Schema Alignment** — The `GrowthbookExperimentEvent` interface maps directly to GrowthBook's SQL experiment assignments table, making it suitable for:
   - Direct database insertion
   - API request/response payloads
   - Event streaming pipelines

3. **Protobuf + JSON Interop** — Provides bidirectional JSON conversion (`fromJSON`/`toJSON`), enabling seamless integration with REST APIs and frontend code that expect plain JSON rather than protobuf binary format.

4. **Full Optionality** — All interface fields use optional (`?:`) syntax. This allows creating events incrementally or with only the fields needed for a specific use case.

5. **Versioned API Contract** — Being under `v1/`, this represents a stable, versioned contract. Future breaking changes would introduce a `v2/` directory.
