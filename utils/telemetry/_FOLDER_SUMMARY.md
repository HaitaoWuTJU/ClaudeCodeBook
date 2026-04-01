# Summary of `utils/telemetry/`

## Purpose of `telemetry/`

The `telemetry/` directory provides a comprehensive observability system for Claude Code, enabling detailed tracking of user interactions, LLM requests, tool executions, and plugin/skill usage via OpenTelemetry. It supports two distinct telemetry backends — OpenTelemetry (with BigQuery export) and Perfetto — and exposes both low-level primitives and high-level abstractions for emitting spans, metrics, and events throughout the codebase.

---

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Public API surface; re-exports all public functions from the module |
| `events.ts` | Low-level event logging: `logOTelEvent()` for emitting telemetry events with attributes, `redactIfDisabled()` for user prompt redaction |
| `sessionTracing.ts` | High-level span API: creates/ends interaction spans, LLM request spans, tool spans, hook spans; manages AsyncLocalStorage context and Perfetto dual-tracing |
| `bigqueryExporter.ts` | Metrics exporter implementing `PushMetricExporter`; transforms OTel metrics to internal format and POSTs to BigQuery API with auth and opt-out handling |
| `pluginTracking.ts` | Domain-specific analytics for plugins: tracks plugin loads, skill loads/executions, handles plugin identity hashing for anonymization |
| `skillLoadedEvent.ts` | Session-startup event emitter; logs `tengu_skill_loaded` for each available prompt-type skill |
| `perfettoTracing.ts` | (referenced) Alternative Perfetto-based tracing backend; co-exists with OpenTelemetry in `sessionTracing.ts` |
| `betaSessionTracing.ts` | (referenced) Beta/experimental tracing attributes for detailed session analysis; injected into spans when beta tracing is enabled |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                        public API                              │
│                      (index.ts exports)                         │
└─────────────────────────────┬───────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌───────────┐      ┌───────────┐      ┌────────────┐
    │  events   │      │   session │      │  plugin    │
    │   .ts     │      │ tracing   │      │ tracking   │
    └─────┬─────┘      └─────┬─────┘      └──────┬─────┘
          │                  │                   │
          ▼                  ▼                   ▼
    logOTelEvent()    Span/Events API      logEvent()
          │                  │                   │
          │         ┌────────┴────────┐          │
          │         │ perfettoTracing │          │
          │         └────────┬────────┘          │
          │                  │                   │
          └──────────────────┼───────────────────┘
                             ▼
                    OpenTelemetry SDK
                             │
                             ▼
                   ┌─────────────────┐
                   │ bigqueryExporter│
                   │    .ts          │
                   └────────┬────────┘
                            ▼
                   POST → BigQuery API
```

**Dependency hierarchy:**

- **`index.ts`** — Re-exports from all public submodules; the single entry point for telemetry consumers
- **`sessionTracing.ts`** — Orchestrates the full tracing lifecycle. It:
  - Calls `events.ts` → `logOTelEvent()` for hook span events (beta)
  - Imports `betaSessionTracing.ts` attributes to inject into spans
  - Optionally bridges to `perfettoTracing.ts` for dual-backend tracing
  - Delegates metrics export to the **sdk-metrics** pipeline, which pipes into `bigqueryExporter.ts`
- **`events.ts`** — Stateless utility; called by `sessionTracing.ts` and `pluginTracking.ts` to emit named events with structured metadata
- **`pluginTracking.ts`** — Calls both `events.ts` (`logOTelEvent()`) and analytics (`logEvent()`) for plugin telemetry; uses `sessionTracing.ts` utilities for plugin span management
- **`skillLoadedEvent.ts`** — Calls analytics (`logEvent()`) directly at session startup; does not flow through `sessionTracing.ts` because it's a session-init event, not a span-bound event
- **`bigqueryExporter.ts`** — Lives at the bottom of the pipeline; receives `ResourceMetrics` from the OpenTelemetry SDK and forwards them (authenticated, transformed) to BigQuery. It does not import other telemetry files — it is a leaf consumer.

---

## Key Takeaways

### Dual Tracing Architecture

`sessionTracing.ts` can run both OpenTelemetry and Perfetto tracing simultaneously. OpenTelemetry flows through `bigqueryExporter.ts` to BigQuery; Perfetto provides local trace file output. This allows Anthropic to debug sessions internally (Perfetto) while shipping telemetry data externally (BigQuery).

### Abstraction Levels

The module exposes three abstraction tiers:

| Tier | Example | Who uses it |
|------|---------|-------------|
| Low-level | `logOTelEvent(span, name, attrs)` | Internal telemetry modules |
| Mid-level | `startToolSpan()` / `endToolSpan()` | Tool runner, hook executor |
| High-level | `startInteractionSpan()` wrapping full workflow | CLI entry point |

### Content Safety & PII Controls

- **`OTEL_LOG_USER_PROMPTS`** env var controls whether user prompts appear in telemetry at all
- **`redactIfDisabled()`** in `events.ts` provides a single enforcement point
- `pluginTracking.ts` uses deterministic hashing for plugin names (prevents telemetry leakage of third-party plugin identifiers)
- BigQuery uses privileged/redacted column routing (`_PROTO_` prefix) for sensitive fields

### Incremental & Deduplicated Telemetry

- **`betaSessionTracing.ts`** tracks `lastReportedMessageHash` per `querySource` to send only NEW context since the last request — preventing trace payloads from growing linearly with conversation length
- System prompts and tool schemas are hashed and deduplicated (`seenHashes` Set) to avoid logging the same 60KB payload twice per session

### Metrics vs. Events vs. Spans

| Signal type | Backend | File |
|-------------|---------|------|
| **Metrics** (counters, histograms) | BigQuery via `bigqueryExporter.ts` | `bigqueryExporter.ts` |
| **Events** (point-in-time signals) | OpenTelemetry pipeline → BigQuery | `events.ts` |
| **Spans** (intervals with hierarchy) | OTel spans → BigQuery; Perfetto spans → local trace | `sessionTracing.ts` |

### Startup vs. Runtime Events

- **`skillLoadedEvent.ts`** fires at session initialization — it is not span-bound and uses analytics `logEvent()` directly
- **`pluginTracking.ts`** fires both at load time (plugin enabled) and at runtime (skill executed), mixing analytics events and span events

### Feature-Gated Telemetry

Both `sessionTracing.ts` and `bigqueryExporter.ts` gate their behavior behind GrowthBook feature flags (`tengu_trace_lantern`) and environment checks, ensuring beta features are controlled before shipping broadly.
