# Summary of `components/shell/`

## Purpose of `shell/`

The `shell/` directory contains a suite of React components and contexts for rendering terminal/shell command output in an Ink-based (React for terminals) application. It handles real-time progress display, ANSI escape code processing, JSON formatting, URL linking, content truncation, and timing information.

## Contents Overview

| File | Type | Purpose |
|------|------|---------|
| `ExpandShellOutputContext.js` | Context | Boolean flag signaling full (non-truncated) output display for the most recent user `!` command |
| `OutputLine.tsx` | Component | Renders individual shell output lines with JSON formatting, URL hyperlink conversion, ANSI cleanup, and conditional truncation |
| `ShellProgressMessage.tsx` | Component | Displays real-time shell progress: last N lines, elapsed time, line counts, file sizes |
| `ShellTimeDisplay.tsx` | Component | Renders elapsed time and/or timeout as dimmed terminal text |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                    ExpandShellOutputContext                  │
│                         (boolean context)                    │
└─────────────────────────┬───────────────────────────────────┘
                          │ useExpandShellOutput()
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                         OutputLine                           │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐    │
│  │ JSON format │→ │ URL linkify  │→ │ ANSI strip/trim  │    │
│  └─────────────┘  └──────────────┘  └───────────────────┘    │
│  Uses: columns, expandShellOutput, inVirtualList, verbose    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   ShellProgressMessage                        │
│  ┌──────────────────┐  ┌─────────────────┐  ┌────────────┐  │
│  │ stripAnsi()      │→ │ slice last 5    │  │ ShellTime  │  │
│  │ (output cleanup) │  │ lines if not    │→ │ Display    │  │
│  │                  │  │ verbose         │  │ (timing)   │  │
│  └──────────────────┘  └─────────────────┘  └────────────┘  │
│  Wrapped in OffscreenFreeze to prevent scroll resets        │
└─────────────────────────────────────────────────────────────┘
```

### Shared Dependencies

| Utility | Source | Used By |
|---------|--------|---------|
| `formatDuration`, `formatFileSize` | `utils/format.js` | `ShellProgressMessage`, `ShellTimeDisplay` |
| `renderTruncatedContent` | `utils/terminal.js` | `OutputLine` |
| `jsonParse`, `jsonStringify` | `utils/slowOperations.js` | `OutputLine` |
| `Ansi`, `Text`, `Box` | `ink.js` | All components |
| `MessageResponse` | `components/MessageResponse.js` | `OutputLine`, `ShellProgressMessage` |
| `useTerminalSize` | `hooks/useTerminalSize.js` | `OutputLine` |

## Key Takeaways

1. **Truncation Strategy**: The system uses a multi-level truncation approach:
   - `ShellProgressMessage` limits to last 5 lines by default
   - `OutputLine` further truncates based on terminal columns and virtual list context
   - Context `expandShellOutput` overrides truncation for the latest user command

2. **React Compiler Optimization**: All components use `react/compiler-runtime` (`_c`) and explicit memoization caches (`$` arrays), indicating compilation with React Compiler for automatic optimization.

3. **Ink Framework**: All UI rendering uses Ink (React for terminals) components (`Box`, `Text`, `Ansi`) rather than standard HTML.

4. **ANSI Handling**: The code explicitly handles ANSI escape codes with multiple strategies:
   - Stripping underline codes (known leak bug)
   - Converting to styled `<Ansi>` components
   - Full `stripAnsi()` for progress message display

5. **Progress UI Pattern**: `OffscreenFreeze` wraps `ShellProgressMessage` to prevent terminal scroll resets during frequent updates—this is a documented workaround for BashTool progress tracking.
