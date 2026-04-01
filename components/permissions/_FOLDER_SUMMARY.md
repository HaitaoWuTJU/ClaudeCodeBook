# Summary of `components/permissions/`

## Purpose of `permissions/`

The `permissions/` directory is the **permission orchestration layer** for a CLI-based AI coding assistant (similar to Claude Code). It provides a unified system for presenting interactive permission dialogs to users whenever the AI attempts to perform actions that require explicit consent — such as executing bash commands, reading/writing files, browsing the web, controlling the computer, or using external tools and skills.

The directory implements a **plugin-like architecture** where each action type (bash, file read/write, web fetch, etc.) has its own dedicated permission request component. The parent orchestrator (`PermissionRequest.js`) dispatches to the correct component based on the tool name, while shared infrastructure (`PermissionDialog`, `PermissionPrompt`, hooks, types) is reused across all permission types.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PermissionRequest.jsx                           │
│                    (orchestrator / dispatcher)                            │
│                                                                          │
│  ┌──────────────────────┐    ┌──────────────────────┐                    │
│  │ tool.name === "bash" │───►│ BashPermissionRequest│                    │
│  └──────────────────────┘    └──────────────────────┘                    │
│  ┌──────────────────────────┐  ┌──────────────────────────┐            │
│  │ tool.name === "mcp..."    │─►│ ComputerUseApproval       │            │
│  └──────────────────────────┘  └──────────────────────────┘            │
│  ┌──────────────────────────┐  ┌──────────────────────────┐            │
│  │ tool.name === "skill"     │─►│ SkillPermissionRequest   │            │
│  └──────────────────────────┘  └──────────────────────────┘            │
│  ┌──────────────────────────┐  ┌──────────────────────────┐            │
│  │ tool.name === "web_fetch"│─►│ WebFetchPermissionRequest│            │
│  └──────────────────────────┘  └──────────────────────────┘            │
│  ┌────────────────────────────┐  ┌──────────────────────────┐          │
│  │ tool.name === "ask_question"│─►│ AskUserQuestionPermission │          │
│  └────────────────────────────┘  └──────────────────────────┘          │
│  ┌────────────────────────────┐  ┌──────────────────────────┐          │
│  │ tool.name === "enter_plan" │─►│ EnterPlanModePermission  │          │
│  └────────────────────────────┘  └──────────────────────────┘          │
│                    ...more tool types...                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

## Contents Overview

| File / Directory | Role |
|---|---|
| **`PermissionRequest.jsx`** | Main orchestrator. Receives `ToolUseConfirm`, dispatches to the correct tool-specific component. Implements FallbackPermissionRequest and the core `Select`/`SelectMulti` question rendering. |
| **`PermissionDialog.jsx`** | Shared layout shell (title, rule explanation, body slot, footer with options). Wraps all permission dialogs. |
| **`PermissionPrompt.jsx`** | Shared prompt component rendering a `Select` with labeled options and optional Haiku-style subtitle. |
| **`PermissionRuleExplanation.tsx`** | Renders a formatted explanation of what permission rule(s) will be created by the user's choice. |
| **`hooks.ts`** | Exports `usePermissionRequestLogging` (logs analytics events, updates `attribution.permissionPromptCount`) and helper utilities. |
| **`utils.ts`** | Exports `logUnaryPermissionEvent()` for unified unary analytics logging. |
| **`WorkerBadge.tsx`** | Renders a colored badge (`@name`) for swarm worker identification in permission prompts. |
| **`WorkerPendingPermission.tsx`** | Shows a "waiting for team lead approval" spinner panel for swarm workers. |
| **`useShellPermissionFeedback.ts`** | Manages yes/no feedback-mode state, focus tracking, and rejection with analytics for shell tools. |
| **`PermissionDecisionDebugInfo.tsx`** | Debug-only panel showing permission decision reasons, unreachable rules, and suggested fixes. |
| **`AskUserQuestionPermissionRequest/`** | Multi-question questionnaire UI with syntax-highlighted preview, image paste support, and submit/review flow. |
| **`BashPermissionRequest/`** | Bash command permission UI with destructive warning banners, sandbox badges, and classifier-suggested "always allow" rules. |
| **`ComputerUseApproval/`** | macOS TCC (Accessibility/Screen Recording) permission checks + app allowlist with sentinel-category warnings. |
| **`EnterPlanModePermissionRequest/`** | Simple dialog confirming entry into plan mode before code changes. |
| **`FilePermissionRequest/`** | File read/write permission UI with diff preview, directory tree, syntax highlighting, and workspace management tabs. |
| **`SedEditPermissionRequest/`** | Sed-based file substitution preview, delegating to `FilePermissionDialog` after computing the edited content. |
| **`SkillPermissionRequest/`** | Skill/tool execution permission with exact-match, prefix, and base granularity for persistent "always allow" rules. |
| **`WebFetchPermissionRequest/`** | Web fetch permission dialog showing URL, domain scope, and one-time vs. permanent allow options. |

## How Files Relate to Each Other

### Permission Flow

```
toolUseConfirm (from parent agent/tool layer)
    │
    ▼
PermissionRequest.jsx (orchestrator)
    │ Inspects tool.name
    │
    ├─ "bash" ──► BashPermissionRequest/BashPermissionRequest.tsx
    │                   ├─ useShellPermissionFeedback() [hooks.ts]
    │                   ├─ usePermissionExplainerUI()
    │                   ├─ generateGenericDescription()
    │                   └─ BashPermissionRequest/bashToolUseOptions.tsx
    │                           └─ PermissionRuleExplanation.tsx
    │
    ├─ "mcp::computer_use" ──► ComputerUseApproval/index.tsx
    │                               └─ uses TCC types, sentinel app classifier
    │
    ├─ "file_read"/"file_write" ──► FilePermissionRequest/
    │                                   ├─ PermissionDialog (shared)
    │                                   ├─ PermissionRuleExplanation (shared)
    │                                   └─ FileBrowserTab, WorkspaceTab, RecentDenialsTab
    │
    ├─ "web_fetch" ──► WebFetchPermissionRequest/
    │                       └─ PermissionDialog (shared)
    │
    ├─ "skill" ──► SkillPermissionRequest/
    │                   └─ PermissionDialog, PermissionPrompt (shared)
    │
    ├─ "ask_question" ──► AskUserQuestionPermissionRequest/
    │                         ├─ PreviewBox, PreviewQuestionView
    │                         ├─ SubmitQuestionsView
    │                         └─ QuestionNavigationBar
    │
    ├─ "sed_edit" ──► SedEditPermissionRequest/
    │                      └─ FilePermissionDialog (delegates)
    │
    ├─ "enter_plan" ──► EnterPlanModePermissionRequest/
    │                        └─ PermissionDialog (shared)
    │
    └─ (unknown tool) ──► FallbackPermissionRequest (inline in PermissionRequest.jsx)
                              ├─ renders permission result rule explanation
                              └─ logs via usePermissionRequestLogging() [hooks.ts]
```

### Shared Infrastructure Wiring

```
hooks.ts (usePermissionRequestLogging)
    │ Logs: logUnaryEvent, logEvent, attribution counts
    │ ▲
    │ └──────────────────┐
    │
PermissionDialog.jsx
    │ Renders: title, rule explanation slot, body, options footer
    │ ▲
    │ └──────────────────┐
    │
PermissionRuleExplanation.tsx
    │ Formats: PermissionRule → human-readable string (rule content + behavior + source)
    │ Used by: File, Bash, Skill, WebFetch, SedEdit, Fallback
    │
PermissionPrompt.jsx
    │ Renders: Select with labeled options + optional subtitle
    │ Used by: Bash, Skill, WebFetch, Fallback
    │
useShellPermissionFeedback.ts
    │ Manages: yes/no feedback mode, focus, reject tracking
    │ Used by: BashPermissionRequest
    │
WorkerBadge.tsx + WorkerPendingPermission.tsx
    │ Renders: colored worker badge + pending spinner
    │ Used by: any permission component in swarm mode
    │
PermissionDecisionDebugInfo.tsx
    │ Debug panel: shows decision reason + unreachable rules
    │ Used by: FilePermissionRequest (workspace management)
```

## Key Takeaways

### Architecture Patterns

1. **Plugin dispatch model** — `PermissionRequest.jsx` is the single entry point; it inspects `tool.name` and renders the appropriate sub-component. New tool types only need a new subdirectory and a new `case` in the switch — no changes to shared infrastructure.

2. **Shared layout, custom content** — Every permission dialog uses `PermissionDialog` as the shell and `PermissionRuleExplanation` for the rule display. Tool-specific components only implement the body (the interactive options), keeping UI patterns consistent.

3. **Rule-based permission system** — Permissions are not simple booleans. They are structured rules with `toolName`, `ruleContent`, `behavior` (allow/ask/deny), and `destination` (localSettings/projectSettings). This enables persistent "always allow" rules that survive restarts.

4. **Analytics-first design** — Every permission event (accept, reject, feedback entered, dialog opened/closed) is logged via `usePermissionRequestLogging` → `logUnaryEvent` / `logEvent`, with `message_id`, `platform`, `completion_type`, and `language_name` metadata for attribution.

### Permission Granularity

| Tool | One-time | Persistent | Granularity |
|------|----------|------------|-------------|
| Bash | ✅ | ✅ (exact, prefix, base) | command pattern + working dir |
| File Read/Write | ✅ | ✅ (exact path, directory) | path prefix |
| Web Fetch | ✅ | ✅ (domain) | hostname |
| Skill | ✅ | ✅ (exact, prefix, base) | command name |
| Sed Edit | ✅ (preview) | via FilePermissionDialog | file path |
| Computer Use | ✅ (per app) | ✅ (per app) | bundle ID |
| Ask Question | ✅ | N/A | N/A |
| Enter Plan Mode | ✅ | N/A | N/A |

### React Compiler Adoption

All components across the entire directory use `react/compiler-runtime` with manual `$` cache arrays (slots indexed `$[0]` through `$[57]`) and `Symbol.for("react.memo_cache_sentinel")` as memoization sentinels. This indicates a codebase actively migrated to React Compiler optimization — components get fine-grained memoization without explicit `useMemo`/`useCallback` hooks, but with explicit control over which values are cached.

### Shared Patterns

| Pattern | Files Using It |
|---------|----------------|
| `_c()` + `$` cache array + sentinel | All components |
| `PermissionDialog` + `PermissionPrompt` | Bash, Skill, WebFetch, Fallback |
| `usePermissionRequestLogging` | Fallback, AskUserQuestion, File |
| `logUnaryPermissionEvent` | Skill, WebFetch, Fallback |
| `WorkerBadge` | All (for swarm mode) |
| `detectUnreachableRules` | FilePermissionRequest, BashToolUseOptions |
| `shouldShowAlwaysAllowOptions()` guard | Bash, Skill, WebFetch, File |
| Ink.js (`Box`, `Text`, `Select`, `Ansi`, `useTheme`) | All components (terminal UI) |

### Security Considerations

- **Destructive command warnings** — Bash tool checks for destructive commands (rm, dd, mkfs, etc.) and shows warning banners; feature-gated by `tengu_destructive_command_warning`.
- **Sandbox isolation** — Bash tool shows a sandbox badge when `SandboxManager.isSandboxingEnabled()`; sandboxed commands may bypass destructive command warnings.
- **Sentinel app classification** — ComputerUseApproval classifies requested apps into `shell`, `filesystem`, and `system_settings` risk categories with visual warnings.
- **TCC permission checks** — ComputerUseApproval checks macOS Accessibility and Screen Recording permissions before allowing screen capture.
- **Classifier as guard** — `isClassifierPermissionsEnabled` gates the auto-approval "classifier-reviewed" option, ensuring ML-based decisions are explicit and user-visible.
- **Unreachable rule detection** — When users add rules, `detectUnreachableRules()` warns about existing rules that would shadow the new one, preventing silent permission failures.
