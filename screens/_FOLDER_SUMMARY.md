# Summary of `screens/`

## Purpose of `screens/`

The `screens/` directory contains the primary **interactive UI surfaces** for the Claude Code CLI — each representing a distinct user-facing mode or entry point. Together they form the complete user journey:

```
Startup / Bootstrap
       │
       ├──► REPL (main interactive loop)
       │         │
       │         ├──► Sandbox / Agent screens (via internal navigation)
       │         └──► Doctor diagnostics (via command: 'claude doctor')
       │
       └──► ResumeConversation (via command: 'claude resume')
                 │
                 └──►► REPL (with restored session state)
```

---

## Contents Overview

| File | Size | Role | Primary User Flow |
|---|---|---|---|
| **`REPL.tsx`** | ~895 KB | Core interactive terminal loop — message rendering, tool execution, permissions, multi-agent coordination | Active conversation session |
| **`Doctor.tsx`** | ~35 KB | System diagnostics — version checks, path validation, MCP parse errors, plugin errors, env var warnings | `claude doctor` command |
| **`ResumeConversation.tsx`** | ~27 KB | Conversation browser — search, filter by PR, select and restore a past session | `claude resume` command |

---

## How Files Relate to Each Other

### Navigation Paths

```
REPL
 ├── "confirm:yes" / "confirm:no" keybinding
 │         │
 │         └──► Doctor (full-screen overlay, dismissable)
 │
 └── session.end / session.done events
           │
           └──► ResumeConversation (session selector)

ResumeConversation
 └── onSelect(log) → async handler
           │
           ├──► REPL (new screen mount with loaded session state)
           │
           └──► CostTracker, analytics events

Doctor
 └── onDone() → caller-defined callback
           │
           └──► Back to calling screen (REPL, config flow, etc.)
```

### Shared Infrastructure

All three screens depend on the same foundation:

| Shared Module | Used By |
|---|---|
| **`src/bootstrap/state.js`** — Global app state (agent, session, tools) | All three |
| **`src/keybindings/useKeybinding.js`** — Keyboard binding system | All three |
| **`src/ink.js`** — Ink component registry | All three |
| **`src/hooks/useExitOnCtrlCDWithKeybindings.js`** | Doctor, ResumeConversation |
| **`src/utils/sessionStorage.js`** | REPL (primary), ResumeConversation (reads) |
| **`src/services/mcp/types.js`** | Doctor, REPL |
| **`src/hooks/useSettingsErrors.js`** | Doctor |
| **`src/utils/agenticSessionSearch.js`** | ResumeConversation |
| **`src/cost-tracker.js`** | REPL (primary), ResumeConversation (restores) |

### State Transitions

```
REPL ──────[session.end]──────► ResumeConversation
    ◄────[session.switchSession]─────│
                                    │
    ◄────[dismiss Doctor]────────────┤
                                    │
Doctor ◄──[app:interrupt]───────────┘
```

---

## Key Takeaways

1. **Screen-driven architecture** — Rather than a single monolithic component, the CLI is organized as discrete, focused screens that mount and unmount based on user intent and system events.

2. **Shared terminal UI layer** — All screens use **Ink** (`ink Box`, `ink Text`) for rendering, ensuring a consistent terminal-native look and feel. React 19's compiler runtime (`react/compiler-runtime`) with `_c(n)` slots is used across all files for optimized memoization.

3. **Session as first-class concept** — `REPL.tsx` treats sessions as core state objects (`switchSession`, `sessionStorage`, `sessionRestore`). `ResumeConversation.tsx` provides the UI for navigating and switching between them.

4. **Feature-flagged compilation** — `REPL.tsx` uses conditional `require()` calls gated by `feature()` and build target checks (`external !== 'ant'`) to strip ANT-only features (voice, frustration detection, coordinator mode) from external builds.

5. **Async initialization pattern** — All three screens perform substantial async work on mount (loading logs, fetching dist-tags, initializing MCP clients) using `useEffect`, async state setters, and `Suspense` boundaries.

6. **Diagnostics are self-contained** — `Doctor.tsx` is intentionally decoupled; it reads the app state it needs via hooks but doesn't mutate global state, making it safe to call from anywhere in the flow.

7. **Analytics integration** — `REPL.tsx` and `ResumeConversation.tsx` both emit structured analytics events (`logEvent`/`tengu_session_resumed`) for telemetry, while `Doctor.tsx` focuses on local feedback rather than analytics.
