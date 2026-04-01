# Summary of `components/HelpV2/`

## Purpose of `HelpV2/`

Provides a modular, tabbed help dialog system for a CLI application (Claude Code), enabling users to browse general information and command documentation directly in the terminal.

## Contents Overview

| File | Role | Lines |
|------|------|-------|
| `HelpV2.tsx` | Root component — orchestrates tabs, filters commands, registers keyboard shortcuts | 180 |
| `General.tsx` | Tab content — static introductory text describing the CLI's capabilities | 55 |
| `Commands.tsx` | Tab content — interactive command browser with keyboard navigation | 147 |

## How Files Relate to Each Other

```
HelpV2.tsx (orchestrator)
├── uses Tab: "General"
│   └── renders General.tsx → descriptive text + PromptInputHelpMenu
│
└── uses Tab: "Commands" + "Custom"
    └── renders Commands.tsx → Select list
        ├── reads commands prop (passed from parent)
        ├── filters into built-in / custom
        ├── computes maxHeight from terminal rows
        └── renders Select component with truncation
```

**Data flow**: `HelpV2` receives the full `commands` array → filters it by `builtInCommandNames()` → passes the appropriate subsets to `Commands` as tab content. `General` contains no dynamic data.

## Key Takeaways

- **Tab-based navigation** — The help system uses a `Tabs` design-system component with three sections: General, Built-in Commands, and Custom Commands
- **Terminal-aware layout** — Both `HelpV2` and `Commands` read terminal dimensions via `useTerminalSize()` to constrain height
- **Command deduplication & sorting** — `Commands` removes duplicate names and sorts alphabetically, ensuring collision-free React keys and a consistent UX
- **Ink library** — All components render using Ink (`Box`, `Text`, `Link`), a React variant for CLI rendering
- **React Compiler** — All three files use `_c` from `react/compiler-runtime`, indicating this project uses React's experimental compiler for automatic memoization
- **Keyboard shortcuts** — `HelpV2` registers `help:dismiss` and provides context-aware exit hints based on `useExitOnCtrlCDWithKeybindings()`
- **Ant-only feature** — A third tab (`[ant-only]`) is defined but disabled via `if (false && ...)`, suggesting a conditional CLI variant
