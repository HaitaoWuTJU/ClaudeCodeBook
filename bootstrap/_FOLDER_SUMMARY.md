# Summary of `bootstrap/`

## Purpose of `bootstrap/`

The `bootstrap/` directory contains **application initialization and foundational setup** logic for the CLI tool. It establishes the core runtime environment, initializes global state, and configures essential services (telemetry, logging, session management) before the main application logic executes.

## Contents Overview

| File | Purpose |
|------|---------|
| `state.ts` | Central state management singleton; defines `State` interface with ~80 fields for tracking API costs, model usage, telemetry, plugins, hooks, session IDs, caching, beta feature flags, and cron tasks; provides getter/setter functions for all state access |
| *(likely other bootstrap files)* | Application entry point coordination, service initialization, and pre-flight checks |

## How Files Relate to Each Other

```
bootstrap/
├── state.ts          ← Initializes STATE singleton via getInitialState()
│   ├── Reads cwd, resolves symlinks
│   ├── Generates session UUID
│   └── Exposes typed accessors for all app state
│
└── (other files)     ← Import STATE for runtime access
    │
    └── Other modules
        ├── api/      ← Increment cost/duration counters
        ├── tools/    ← Register hooks, track invoked skills
        ├── telemetry/← Access Meter/Logger/Tracer providers
        └── commands/ ← Read/write session, settings, flags
```

**Data Flow:**
1. Bootstrap phase → `getInitialState()` creates `STATE` with resolved paths and UUID
2. Telemetry layer → `initMeter/Logger/Tracer` attaches OTel providers to `STATE`
3. Runtime → All modules import and mutate `STATE` via typed accessor functions
4. Commands/plugins → Query `STATE` for session context, settings, cached content

## Key Takeaways

- **`state.ts` is the backbone** of runtime data flow — every API call, tool execution, and hook invocation touches this state
- **Type-safe access pattern**: All state mutations go through named functions (`addModelUsage()`, `registerHookCallbacks()`) rather than direct field assignment
- **Telemetry-first design**: Built-in support for Meter, Logger, and Tracer providers from startup
- **Sticky beta flags**: Header latches (e.g., `afkMode`, `fastMode`) persist once activated to prevent prompt cache invalidation mid-session
- **Plugin extensibility**: Dual hook system supports both SDK callbacks and native plugin hooks via `RegisteredHookMatcher` union type
- **Performance instrumentation**: Ring buffer for slow operations (10-entry, 10s TTL) and in-memory error log for bug reports
- **Session isolation**: UUID-based session tracking with parent session support enables subagent hierarchy and team cleanup
