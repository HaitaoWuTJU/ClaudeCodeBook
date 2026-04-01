# Summary of `utils/permissions/`

## Purpose of `permissions/`

This directory implements Claude Code's **permission system** for agent tool calls, supporting two operating modes:

| Mode | Description |
|------|-------------|
| **Manual** | User is prompted for each permission request |
| **Auto Mode** | A YOLO (security) classifier automatically allows or blocks tool calls based on customizable security rules, without user interaction |

The system is split into **ANT-only** (full-featured) and **external** (stub) implementations, with conditional loading via `bun:bundle` feature flags.

---

## Contents Overview

### Core Permission Infrastructure

| File | Purpose |
|------|---------|
| `permissionSetup.js` | Central coordinator; loads ANT-only modules, initializes permission contexts, provides `setupPermissions()` entry point |
| `permissions.js` | Core `PermissionRule` structures and accessors (`getAllowRules`, `getAskRules`, `getDenyRules`) for reading permission contexts |
| `PermissionUpdateSchema.js` | Zod schemas for `PermissionUpdate` DTOs and rule normalization |

### Auto Mode Classification

| File | Purpose |
|------|---------|
| `yoloClassifier.ts` | **ANT-only** — Core YOLO classifier. Builds system prompts from transcripts + security rules, calls the model via single-stage or two-stage XML classifier, returns `YoloClassifierResult` |
| `bashClassifier.ts` | **ANT-only** — Bash-specific classifier. Matches commands against `ask`/`deny` rules to determine whether to auto-block or prompt |
| `classifierDecision.ts` | Tool allowlist for auto mode (tools like `Read`, `Glob`, `Grep` that never need classification) |
| `classifierShared.ts` | Shared extraction/parsing utilities for ANT and external builds |
| `autoModeState.ts` | Module-level state tracking (`autoModeActive`, `autoModeFlagCli`, `autoModeCircuitBroken`) |

### Shell Rule Matching

| File | Purpose |
|------|---------|
| `shellRuleMatching.ts` | Parses and matches permission rules supporting **exact**, **prefix** (`npm:*`), and **wildcard** (`git *`) patterns with escape sequence support |

### Feature Gating (Killswitches)

| File | Purpose |
|------|---------|
| `bypassPermissionsKillswitch.ts` | Statsig-based gates that disable bypass permissions or auto mode per-organization (via `shouldDisableBypassPermissions` and `verifyAutoModeGateAccess`) |
| `permissionsKillswitch.js` | Top-level exports/killswitch for bypass permissions feature |
| `bashClassifier.ts` | **External stub** — All functions return disabled/empty values for non-ANT builds |

### Rule Analysis

| File | Purpose |
|------|---------|
| `detectUnreachableRules.ts` | Detects allow rules that are shadowed by tool-wide `ask` or `deny` rules, with fix suggestions |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                    permissionSetup.js                       │
│         (entry point, loads ANT/external modules)           │
└────────────┬──────────────────────────────────┬─────────────┘
             │                                  │
    ┌────────▼────────┐               ┌─────────▼──────────┐
    │  ANT-only path  │               │  External path     │
    │ (if TRANSCRIPT_ │               │ (stub fallback)     │
    │  CLASSIFIER)    │               │                    │
    └────────┬────────┘               └─────────────────────┘
             │
    ┌────────▼───────────────────────────────┐
    │         yoloClassifier.ts              │
    │  - classifyYoloAction()                 │
    │  - classifyYoloActionXml()              │
    │  - buildYoloSystemPrompt()              │
    └────────┬───────────────────────────────┘
             │ uses
    ┌────────▼──────────────┐  ┌──────────────▼──────────────┐
    │   bashClassifier.ts   │  │   classifierDecision.ts      │
    │ - classifyBashAction()│  │ - isAutoModeAllowlistedTool()│
    │ - createBashPrompt()   │  └──────────────────────────────┘
    └────────┬──────────────┘
             │ uses
    ┌────────▼────────────────┐  ┌──────────────▼────────────────┐
    │  permissions.js        │  │  autoModeState.ts               │
    │ - getAllowRules()       │  │ - isAutoModeActive()            │
    │ - getAskRules()         │  │ - isAutoModeCircuitBroken()     │
    │ - getDenyRules()        │  └─────────────────────────────────┘
    └────────┬────────────────┘
             │ uses
    ┌────────▼────────────────┐  ┌──────────────▼────────────────┐
    │  shellRuleMatching.ts   │  │  detectUnreachableRules.ts     │
    │ - parsePermissionRule() │  │ - detectUnreachableRules()     │
    │ - matchWildcardPattern()│  └─────────────────────────────────┘
    └─────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              bypassPermissionsKillswitch.ts                  │
│  (independent; reads Statsig gates, writes AppState)        │
└─────────────────────────────────────────────────────────────┘
```

**Data flow through auto mode:**

```
User message / tool call
  │
  ▼
setupPermissions() → PermissionContext
  │
  ├─► classifyYoloAction() ──► buildYoloSystemPrompt() ──► API call
  │                              │
  │                         toCompactBlock() ──► TranscriptEntry[]
  │                              │
  │                         buildTranscriptEntries() ──► Messages[]
  │
  └─► classifyBashCommand() ──► Bash rule matching ──► ALLOW/ASK/DENY
```

---

## Key Takeaways

1. **Dual-mode architecture**: The system is designed for both ANT (full features) and external builds. Feature-flagged `require()` calls with `DCE` constants allow the bundler to tree-shake unused branches entirely.

2. **Two-stage classifier**: For organizations with `tengu_auto_mode_config.twoStageClassifier` enabled, the classifier runs a fast stage followed by a thinking stage, using `cache_control` breakpoints to guarantee cache hits.

3. **Shell rule matching**: Supports three rule types (exact, prefix `:*`, wildcard `*`) with escape sequences (`\*`, `\\`), plus backwards compatibility for legacy `:*/` syntax.

4. **Circuit breaker**: `autoModeCircuitBroken` in `autoModeState.ts` prevents repeated auto mode re-entry attempts after GrowthBook explicitly disables the feature.

5. **Rule shadowing detection**: `detectUnreachableRules.ts` proactively identifies when specific allow rules are blocked by tool-wide ask/deny rules, suggesting fixes to avoid user confusion.

6. **Statsig-based killswitches**: Both bypass permissions and auto mode can be disabled per-organization via Statsig gates (`shouldDisableBypassPermissions`, `verifyAutoModeGateAccess`) without code changes.

7. **Telemetry**: Classifier outcomes (`tengu_auto_mode_outcome`) and errors are logged for monitoring, with request/response dumps available via `CLAUDE_CODE_DUMP_AUTO_MODE`.
