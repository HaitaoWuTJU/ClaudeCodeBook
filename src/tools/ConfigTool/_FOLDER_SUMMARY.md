# Summary of `tools/ConfigTool/`

## Purpose of `ConfigTool/`

The `ConfigTool/` directory implements a tool for getting and setting Claude Code configuration settings (theme, model, permissions, voice, etc.) with permission checks, validation, and terminal UI feedback. It supports both user-level (global) configuration and project-level settings.

## Contents Overview

| File | Purpose |
|------|---------|
| `ConfigTool.ts` | Main tool definition with `call()` logic, permission handling, input/output Zod schemas, and state synchronization |
| `constants.ts` | Exports `CONFIG_TOOL_NAME = 'Config'` for use across the codebase |
| `prompt.ts` | Generates dynamic markdown documentation for supported settings, filtering based on feature flags and user type |
| `supportedSettings.ts` | Registry of all supported settings with metadata (types, descriptions, options, validation hooks) and query utilities |
| `UI.tsx` | React/Ink terminal UI renderers for tool usage, results, and rejection messages |

## How Files Relate to Each Other

```
┌─────────────────────────┐
│  ConfigTool.ts          │  ← Main entry point
│  (call, permissions,    │     - Exports ConfigTool as default
│   isReadOnly, map-      │     - Imports schemas from supportedSettings
│   ToolResultTo...Param) │     - Imports UI renderers from UI.tsx
└───────────┬─────────────┘
            │
    ┌───────┴────────┐
    │                │
    ▼                ▼
┌─────────────┐  ┌─────────┐
│ supported-  │  │ UI.tsx  │
│ Settings.ts │  │         │
│             │  │         │
│ - Exports   │  │ - Imported by ConfigTool.ts
│   SUPPORTED │  │   for render functions
│   _SETTINGS │  │   used in mapToolResult...
│             │  │   ToToolResultBlockParam()
│ - getConfig │  └─────────┘
│ - isSupport │
│ - getPath   │  ┌─────────────┐
│ - getOpt... │  │ constants.ts│
└──────┬──────┘  │             │
       │         │ - Exports   │
       │         │   CONFIG_   │
       │         │   TOOL_NAME  │
       └─────────┴─────────────┘

       ┌─────────────┐
       │ prompt.ts   │
       │             │
       │ - Generates │
       │   markdown  │
       │   docs      │
       │             │
       └─────────────┘
```

**File Relationships:**

1. **`ConfigTool.ts`** orchestrates everything:
   - Uses `getOptionsForSetting()`, `SUPPORTED_SETTINGS` from `supportedSettings.ts`
   - Imports `renderToolUseMessage`, `renderToolResultMessage` from `UI.tsx`
   - References `CONFIG_TOOL_NAME` from `constants.ts`
   - Reads dynamic model options via `getModelOptions()` for prompts

2. **`supportedSettings.ts`** is the central data source:
   - Provides the `SUPPORTED_SETTINGS` registry consumed by ConfigTool and prompt
   - Exports helper functions used by both `ConfigTool.ts` and `prompt.ts`
   - Conditionally includes settings based on feature flags and environment

3. **`prompt.ts`** reads from `supportedSettings.ts`:
   - Iterates through `SUPPORTED_SETTINGS` to build dynamic documentation
   - Uses `isVoiceGrowthBookEnabled()` to filter voice-related settings

4. **`constants.ts`** and **`UI.tsx`** are utility modules:
   - Provide simple string constant and terminal UI rendering functions

## Key Takeaways

- **Centralized registry**: All configurable settings are defined in `supportedSettings.ts`, making it easy to add/modify settings
- **Feature gating**: Settings are conditionally included via feature flags (`bun:bundle`) and GrowthBook kill-switches at both build-time and runtime
- **Dual sources**: Settings can originate from global config (`~/.claude.json`) or project settings (`settings.json`)
- **State synchronization**: Writes trigger immediate `AppState` updates and `settingsChangeDetector` notifications
- **Voice pre-flight validation**: Voice mode changes run extensive checks (auth, recording, dependencies, microphone permission) before persisting
- **Dynamic documentation**: The prompt dynamically reflects available settings, filtering based on runtime conditions
- **Terminal-optimized UI**: Uses Ink (React for CLI) with color-coded messages for success, errors, and warnings
