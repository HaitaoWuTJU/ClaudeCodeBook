# Summary of `components/`

# Tengu CLI — Interactive Terminal UI Framework

## Purpose

This codebase is the interactive shell frontend for **Tengu**, a CLI tool that assists users with AI-powered coding, Git worktree management, team collaboration, and plugin installation. The entire UI is built with **Ink.js** (React for terminals), enabling rich, interactive terminal experiences with menus, dialogs, progress indicators, and keyboard-driven navigation — all rendered as terminal output rather than a browser.

The framework provides:

- **Agent execution infrastructure** — progress tracking, tool call display, permission handling, token metering
- **Wizard and dialog systems** — multi-step creation flows, confirmation dialogs, exit flows
- **Virtualized terminal UI** — efficient rendering of long chat histories and large code outputs
- **Keyboard-first interaction** — comprehensive keyboard shortcuts, configurable hints, modal dialogs
- **State management** — React Context providers for app state, stats, FPS metrics, and wizard navigation

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Entry Points                                 │
├──────────────────────────────────────────────────────────────────────┤
│  App.tsx                    AgentProgressLine.tsx                     │
│  (root provider stack)     (agent execution rendering)                │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────┴───────────────────────────────────┐    │
│  │                  Core Provider Stack                          │    │
│  │                                                               │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │    │
│  │  │ FpsMetrics      │  │ Stats           │  │ AppState    │  │    │
│  │  │ Provider        │  │ Provider        │  │ Provider    │  │    │
│  │  │ (outermost)     │  │                 │  │ (innermost) │  │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────┘  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────┴───────────────────────────────────┐    │
│  │                  Component Modules                            │    │
│  │                                                               │    │
│  │  Wizard/           Design-system/       CustomSelect/        │    │
│  │  ├─ WizardProvider │ ├─ Dialog.tsx      ├─ select.tsx        │    │
│  │  ├─ WizardDialog... │ ├─ Byline.tsx      ├─ SelectMulti.jsx  │    │
│  │  └─ WizardNavig...  │ ├─ ColorPicker.jsx ├─ select-input... │    │
│  │                     │ ├─ Permission...   └─ use-select...   │    │
│  │  agents/            │ └─ ThemedText.tsx                       │    │
│  │  ├─ new-agent-...   │                                          │    │
│  │  └─ AgentProgress... │  ClaudeCodeHint/                        │    │
│  │                     │  ├─ Index.jsx                           │    │
│  │  TrustDialog/       │  └─ index.ts                            │    │
│  │  ├─ TrustDialog.tsx │                                          │    │
│  │  └─ utils.ts        │  PluginHintMenu/                         │    │
│  │                     │  ├─ index.ts                            │    │
│  │  WorktreeExitDialog │  └─ PluginHintMenu.jsx                  │    │
│  │  └─ WorktreeExit... │                                          │    │
│  │                     │  TeamMembersPanel/                       │    │
│  │  ApproveApiKey.tsx  │  ├─ TeamDetailView.tsx                   │    │
│  │                     │  └─ TeammateListItem.tsx                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────┴───────────────────────────────────┐    │
│  │                  Ink.js (React for CLI)                       │    │
│  │                                                               │    │
│  │  Box, Text, Newline, Spacer, Measure, useFocus, useInput    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Contents Overview

### Provider Layer (`App.tsx`)

| Provider | Purpose |
|----------|---------|
| `FpsMetricsProvider` | Exposes render performance metrics via `getFpsMetrics()` |
| `StatsProvider` | Distributes the global stats store |
| `AppStateProvider` | Manages top-level app state with `onChangeAppState` handler |

### Wizard System (`wizard/`)

| Component | Purpose |
|-----------|---------|
| `WizardProvider` | Central state manager — owns step index, accumulated data, navigation history |
| `useWizard<T>()` | Hook for children to access wizard context (state + navigation functions) |
| `WizardDialogLayout` | Dialog wrapper rendering the current step with header |
| `WizardNavigationFooter` | Footer with keyboard shortcut hints and exit confirmation |

### Agent System (`agents/`)

| Component | Purpose |
|-----------|---------|
| `AgentProgressLine` | Renders a tree-style progress line with status, tool count, token count |
| `new-agent-creation/` | Full wizard flow for creating agents (type → model → tools → memory → color → location → summary) |

### Dialogs

| Dialog | Purpose |
|--------|---------|
| `TrustDialog` | Security gate auditing 7 danger categories before granting file access |
| `ApproveApiKey` | Prompt to approve/reject detected custom API keys |
| `PluginHintMenu` | Offer to install a suggested plugin (auto-dismisses after 30s) |
| `WorktreeExitDialog` | Worktree exit flow checking uncommitted changes and tmux sessions |
| `AutoModeOptInDialog` | Opt-in prompt for automatic permission handling |
| `WorkflowMultiselectDialog` | Multi-select for GitHub workflow installation |

### Design System (`design-system/`)

| Component | Purpose |
|-----------|---------|
| `Dialog` | Standard dialog chrome with title, exit hint, optional footer |
| `Byline` | Secondary caption text below titles |
| `ColorPicker` | Terminal-rendered color swatch selector |
| `PermissionDialog` | Specialized dialog for yes/no/disable permission questions |
| `ThemedText` | Text with theme-aware hover coloring |

### Custom Select (`CustomSelect/`)

| Component | Purpose |
|-----------|---------|
| `Select` | Terminal-native dropdown with keyboard navigation and search |
| `SelectMulti` | Multi-item selection with checkboxes |
| `SelectInput` | Text input within select (for inline editing or pasting) |
| `use-select-*` | Composeable hooks for state, navigation, and input handling |
| `option-map.ts` | Doubly-linked list map for O(1) bidirectional option traversal |

### Team Collaboration (`TeamMembersPanel/`)

| Component | Purpose |
|-----------|---------|
| `TeamDetailView` | Collapsible team member with status, mode, permission indicators |
| `TeammateListItem` | Row item in team member list with inline action controls |
| `TeammateDetailView` | Expanded teammate detail with mode cycling, output viewing, permission controls |

---

## How the Core Files Relate

### App Initialization (`App.tsx`)

```
App(props: getFpsMetrics, stats, initialState, children)
    │
    ▼
FpsMetricsProvider(value={getFpsMetrics})
    │
    ▼
StatsProvider(value={stats})
    │
    ▼
AppStateProvider(initialState, onChangeAppState)
    │
    ▼
children (rendered inside all three contexts)
```

### Agent Progress Display (`AgentProgressLine.tsx`)

```
AgentProgressLine(props)
    │
    ├── Renders tree character (├ or └) based on isLast
    │
    ├── Agent Type/Name row:
    │   ├── [AGENT] badge (if !hideType)
    │   ├── Agent name (if name provided)
    │   ├── Description (dimmed)
    │   ├── Tool count badge (if toolUseCount > 0)
    │   └── Token count (if tokens !== null)
    │
    └── Status row (if !isAsync || !isResolved):
        ├── ⏿ spinner symbol
        └── Status text (Initializing… / Running in background / Done)
```

### Wizard Flow (`new-agent-creation/`)

```
AgentCreationDialog.tsx
    │
    ▼
CreateAgentWizard.tsx
    ├── Conditionally adds MemoryStep (if isAutoMemoryEnabled)
    └── Wraps in WizardProvider with steps[]
        │
        ▼
NewAgentWizard.tsx (or CreateAgentWithWantedTasks)
    ├── useWizard<AgentWizardData>() — shared context
    └── Maps steps to WizardStepComponent wrappers
        │
        ├── TypeStep ──── wizardData.type
        ├── WhenToUseStep ── wizardData.whenToUse
        ├── ModelStep ──── wizardData.selectedModel
        ├── SystemPromptStep ─ wizardData.systemPrompt
        ├── ToolsStep ──── wizardData.selectedTools
        ├── MemoryStep ─── wizardData.memorySettings (conditional)
        ├── ColorStep ──── wizardData.selectedColor
        ├── LocationStep ── wizardData.location
        │
        ├── SummaryStep ── Real-time finalAgent assembly
        │
        └── ConfirmStepWrapper.tsx (side effects only)
            ├── saveAgentToFile(finalAgent)
            ├── useSetAppState() → add to agent list
            ├── editFileInEditor() (optional)
            ├── logEvent("agent_created", ...)
            └── onComplete(message)
```

### Trust Dialog Flow (`TrustDialog/`)

```
TrustDialog.tsx
    │
    ├── Calls utility functions (from utils.ts):
    │   ├── getHooksSources()           — MCP server hook configurations
    │   ├── getBashPermissionSources()  — .hushlogin, shell env configs
    │   ├── getOtelHeadersHelperSources() — OpenTelemetry helpers
    │   ├── getApiKeyHelperSources()    — API key helpers
    │   ├── getAwsCommandsSources()     — AWS command configurations
    │   ├── getGcpCommandsSources()     — GCP command configurations
    │   └── getDangerousEnvVarsSources() — Dangerous environment variables
    │
    ├── Aggregates all flagged source files
    │
    ├── Renders Dialog with Byline warning + flagged files list
    │
    └── On acceptance:
        ├── Home dir trust → setSessionTrustAccepted (memory only)
        ├── Project trust → saveGlobalConfig({ trusted: cwd })
        └── logEvent("trust_accepted", { sources, location })
```

---

## Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| **React Compiler everywhere** | All components use `react/compiler-runtime` (`_c`, `Symbol.for("react.memo_cache_sentinel")`) for automatic memoization, critical for smooth terminal rendering |
| **Ink.js over blessed/etc.** | Provides React-compatible component model, easier integration with existing React patterns |
| **Context-separated providers** | Three distinct providers (FPS, Stats, AppState) instead of one large context, enabling granular subscription and better tree-shaking |
| **Wizard generic typing** | `useWizard<T extends Record<string, unknown>>()` allows callers to define their own data shape per wizard |
| **Circular dependency lazy-loading** | `WorktreeExitDialog.tsx` uses inline `require()` to break cycle between sessionStorage and commands |
| **Trust scoped to home vs project** | Home directory trusts are session-only; project trusts are persisted to `.claude/settings.json` |
| **Option-map linked list** | CustomSelect uses a doubly-linked list (`option-map.ts`) for O(1) `moveUp`/`moveDown` traversal |
| **Incremental virtual scroll keys** | ChatUI avoids full O(n) key array rebuild on message append by delta-pushing when the prefix matches |
| **Search pre-lowering** | Messages.tsx provides pre-cached lowered search text; `defaultExtractSearchText` is a fallback for custom extractors |
| **AutoMode legally-reviewed copy** | The auto mode description string is flagged as legally reviewed, stored as a module-level constant |

---

## Notable Patterns

### Compound Components
```tsx
<OrderedList>
  <OrderedListItem>Item one</OrderedListItem>
  <OrderedListItem>Item two</OrderedListItem>
</OrderedList>
```

### Wizard State Sharing
```tsx
// Provider creates context
<WizardProvider steps={[...]} onComplete={fn}>
  {children} // can use useWizard() anywhere
</WizardProvider>

// Consumer reads/writes
const { currentStepIndex, wizardData, goNext, goBack } = useWizard();
```

### Auto-dismiss with Stable Callback Ref
```tsx
// Prevents stale closure in timeout
const onResponseRef = useRef(onResponse);
onResponseRef.current = onResponse; // always fresh

useEffect(() => {
  const t = setTimeout(() => onResponseRef.current('no'), 30000);
  return () => clearTimeout(t);
}, []);
```

### Incremental Key Array (Performance)
```tsx
// Avoids O(n) rebuild on append
if (keys.length > 0 && newKeys.startsWith(keys[keys.length - 1])) {
  keys.push(...newKeys.slice(keys.length));
} else {
  keys = newKeys;
}
```
