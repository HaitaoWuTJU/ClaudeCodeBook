# Summary of `components/PromptInput/`

## Purpose of `PromptInput/`

The `PromptInput/` directory contains the core input components and utilities for the command-line interface's prompt input area. It handles user text input, voice input, command history searching, input mode switching, system notifications, and contextual banners.

This is a terminal-based UI built with the **Ink library** (React for CLIs) for the Claude CLI, supporting features like vim mode, voice input, teammate/swarm mode banners, and IDE integration.

---

## Contents Overview

| File | Purpose |
|------|---------|
| `index.tsx` | Barrel export for all PromptInput components |
| `PromptInput.tsx` | Main input component with text, voice, and status integration |
| `PromptInputLine.tsx` | Renders a single line of prompt text with syntax highlighting |
| `PromptInputContainer.tsx` | Orchestrates cursor, input mode, and error handling |
| `Input.tsx` | Low-level input handler with clipboard, history, and vim support |
| `TextInput.tsx` | Styled text input wrapper |
| `HistorySearchInput.tsx` | Search interface for filtering command history |
| `VoiceIndicator.tsx` | Voice input status (recording/processing/idle) with animations |
| `inputModes.ts` | Utilities for bash (`!`) vs prompt mode prefix handling |
| `inputPaste.ts` | Truncates pasted text >10k chars, stores middle portion separately |
| `Notifications.tsx` | Bottom status bar showing IDE connection, token warnings, etc. |
| `useSwarmBanner.ts` | Hook returning contextual banners for agent/team modes |
| `useShowFastIconHint.ts` | Hook showing "/fast" hint once per session |
| `IssueFlagBanner.tsx` | Stubbed banner for issue reporting (ANT-ONLY, disabled) |
| `utils.ts` | Vim mode detection and platform-specific newline instructions |

---

## How Files Relate to Each Other

```
PromptInput.tsx (orchestrator)
│
├── PromptInputContainer.tsx
│   ├── Input.tsx
│   │   ├── TextInput.tsx
│   │   ├── inputModes.ts (prepend/strip ! prefix)
│   │   └── inputPaste.ts (truncate long pasted text)
│   │
│   ├── HistorySearchInput.tsx
│   │   └── TextInput.tsx (filtered history list)
│   │
│   └── PromptInputLine.tsx (prompt text rendering)
│
├── Notifications.tsx
│   ├── IDE connection status
│   ├── Token usage warnings
│   ├── Auto-updater status
│   └── useSwarmBanner.ts (agent mode banners)
│
├── VoiceIndicator.tsx
│   └── Animated states for voice input
│
└── useShowFastIconHint.ts (fast mode hint overlay)

utils.ts (shared utilities for vim mode, newline instructions)
```

**Data Flow Pattern:**

1. **User Input** → `Input.tsx` captures keystrokes, applies mode prefixes (`!`), handles pasting
2. **History Search** → `HistorySearchInput.tsx` overlays when `/` or `?` triggers search
3. **Status Display** → `Notifications.tsx` reads system state (IDE, tokens, updates)
4. **Contextual Banners** → `useSwarmBanner.ts` provides agent/team mode indicators
5. **Voice Input** → `VoiceIndicator.tsx` shows recording/processing state

---

## Key Takeaways

### Architecture Patterns

- **Ink-based CLI**: All components use Ink primitives (`Box`, `Text`, `useInput`) for terminal rendering
- **React Compiler**: Components use `_c` memoization from `react/compiler-runtime` for optimization
- **Feature Flags**: Multiple `feature()` guards (`VOICE_MODE`, `KAIROS`, `ANT`) gate experimental functionality
- **Hooks-based Logic**: Business logic extracted to custom hooks (`useSwarmBanner`, `useShowFastIconHint`) keeping components pure

### Notable Features

| Feature | Implementation |
|---------|-----------------|
| **Command History Search** | `/` or `?` triggers `HistorySearchInput`; filters by substring match |
| **Bash Mode** | `!` prefix switches to bash mode; handled via `inputModes.ts` |
| **Text Truncation** | Pastes >10k chars truncated with `[...Truncated text #N +M lines...]` reference |
| **Vim Mode** | Global config `editorMode: 'vim'` affects key handling |
| **Platform-specific Newlines** | Shift+Enter on macOS Terminal, `\` + Enter elsewhere |
| **Voice Input States** | Animated shimmer (respects `prefersReducedMotion`) |
| **Swarm/Teammate Banners** | Context-aware color-coded headers for different agent modes |

### Input Processing Pipeline

```
Keystroke → Input.tsx
    │
    ├── Check for mode trigger (!)
    ├── Apply vim mode transformations
    ├── Handle history navigation (up/down arrows)
    ├── Manage paste events (truncate via inputPaste.ts)
    │
    ▼
PromptInputContainer.tsx
    │
    ├── Render cursor
    ├── Show error/undo banners
    └── Orchestrate mode-specific overlays
```

### External Integrations

- **IDE Connection**: Shows connection status in `Notifications.tsx`
- **tmux Detection**: `useSwarmBanner.ts` detects tmux environment for contextual instructions
- **Voice Mode**: Full state machine (`idle` → `recording` → `processing`) with animated feedback
