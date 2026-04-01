# Summary of `services/analytics/`

## Purpose of `analytics/`

The `analytics/` directory implements a **multi-sink telemetry pipeline** for Claude Code that:

- Routes events to **Datadog** (external analytics) and a **first-party event logger** (internal)
- Collects and enriches event metadata with environment, process, and model information
- Applies **feature gates** and **killswitches** via GrowthBook dynamic configuration
- Implements **fail-open** design so analytics failures never break core functionality

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Public API (`logEvent()`, `logGrowthBookExperiment()`) and sink attachment |
| `config.ts` | Shared checks: environment/provider logic for disabling analytics |
| `sink.ts` | **Routing layer**: applies sampling, strips proto fields, fan-out to sinks |
| `sinkKillswitch.ts` | Per-sink killswitches (`datadog`, `firstParty`) via GrowthBook config |
| `growthbook.ts` | Feature gate evaluation and dynamic config caching |
| `datadog.ts` | **Datadog sink**: batches logs, sends to Datadog Logs API |
| `firstPartyEventLogger.ts` | **1P sink**: OpenTelemetry SDK with batch processor |
| `firstPartyEventLoggingExporter.ts` | OTLP-compatible exporter for 1P batch endpoint |
| `metadata.ts` | **Data collection**: PII sanitization, MCP tool name redaction, env/context/metrics |
| `segment.ts` | (Legacy) Segment sink removed in favor of direct sink routing |

## How Files Relate to Each Other

```
logEvent(eventName, metadata)
         │
         ▼
┌────────────────────────────┐
│  index.ts                  │
│  → attachAnalyticsSink()   │──── attaches sink to global state
└────────────┬───────────────┘
             │
             ▼
┌────────────────────────────┐
│  sink.ts                   │
│  → logEventImpl()          │
│    ├─ shouldSampleEvent()  │ ◄─── growthbook.ts (sample rates)
│    ├─ shouldTrackDatadog() │ ◄─── sinkKillswitch.ts + growthbook.ts
│    ├─ isSinkKilled()       │ ◄─── sinkKillswitch.ts (killswitch config)
│    └─ stripProtoFields()   │ ◄─── index.ts (remove PII fields)
└────────┬────────┬─────────┘
         │        │
         ▼        ▼
┌──────────────┐  ┌────────────────────────────┐
│  datadog.ts  │  │  firstPartyEventLogger.ts  │
│              │  │                            │
│ trackDatadog │  │ logEventTo1P()             │
│ Event()      │  │   └─→ FirstPartyEventLog   │
│              │  │        gingExporter        │
│ Batches logs │  │                            │
│ Sends to DD  │  │ OTEL BatchLogRecordProcess │
│ API          │  │   └─→ /api/event_logging   │
└──────────────┘  └────────────────────────────┘
                              │
                              ▼
                     ┌────────────────────────┐
                     │  metadata.ts           │
                     │                        │
                     │  getEventMetadata()     │
                     │    ├─ buildEnvContext() │
                     │    └─ buildProcessMetrics()
                     │                        │
                     │  Enriches events with: │
                     │  • Platform/OS/WLS     │
                     │  • Model info          │
                     │  • CPU/memory metrics  │
                     │  • MCP tool redaction  │
                     │  • Agent/session IDs   │
                     └────────────────────────┘
```

**Initialization Order:**
```
index.ts → initializeAnalyticsGates() → growthbook.ts (caches feature flags)
index.ts → initializeAnalyticsSink()  → sink.ts → sinks initialized (datadog, 1P)
```

## Key Takeaways

1. **Fail-open reliability**: Missing config or errors default to logging events rather than silently dropping them, ensuring analytics never blocks core functionality.

2. **Proto field stripping**: `_PROTO_*` keys (containing unredacted PII-tagged values) are stripped before Datadog sends but preserved for 1P logging — 1P has privileged access, Datadog is general-access.

3. **Privacy-preserving techniques**:
   - MCP tool names redacted to `mcp_tool` by default
   - User IDs hashed to 30 buckets for privacy-safe user counting
   - Tool inputs truncated and depth-limited
   - Sampling rates applied via GrowthBook dynamic config

4. **Feature gates via GrowthBook**: All sampling rates, killswitches, and Datadog gate use cached stale reads from GrowthBook, allowing config changes without redeployment.

5. **Graceful shutdown**: Both sinks expose shutdown functions that must be called before `process.exit()` since `forceExit()` prevents `beforeExit` handlers.

6. **Idempotent initialization**: Both `initializeAnalyticsSink()` and `initializeDatadog()` can be called multiple times safely — subsequent calls are no-ops.
