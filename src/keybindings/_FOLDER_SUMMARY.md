# Summary of `keybindings/`

## Purpose of `keybindings/`

The `keybindings/` directory provides a **complete keyboard binding system** for a terminal-based CLI application (Claude Code). It handles loading default bindings, merging user overrides, validating configurations, resolving key inputs to actions, managing chord sequences (multi-key combos), and exposing hooks for components to consume bindings.

## Contents Overview

| File | Purpose |
|------|---------|
| `types.ts` | TypeScript interfaces and types for the binding system |
| `parser.ts` | Parses keystroke strings (`ctrl+c`) into structured data and formats them back for display |
| `defaultBindings.ts` | Default keybindings organized by 20 contexts (Global, Chat, Settings, etc.) with platform-specific and feature-gated bindings |
| `reservedShortcuts.ts` | Lists OS/Terminal reserved shortcuts that cannot be rebound |
| `loadUserBindings.ts` | Loads and parses user bindings from `~/.claude/keybindings.json` with validation |
| `validate.ts` | Validates user configurations: parse errors, duplicates, reserved conflicts, invalid contexts/actions |
| `resolver.ts` | Core resolution logic: maps incoming keystrokes to actions, handles chord sequences, computes display text |
| `KeybindingContext.tsx` | React Context provider exposing binding resolution, handler registration, and chord state management |
| `KeybindingProviderSetup.tsx` | Orchestrates provider setup: loads bindings, manages hot-reload from file changes, wraps with ChordInterceptor |
| `useKeybinding.ts` | Consumer hooks (`useKeybinding`, `useKeybindings`) for components to register action handlers |
| `useShortcutDisplay.ts` | Utility hook to retrieve and display shortcut text for UI labels |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          LOADING & VALIDATION                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  defaultBindings.ts  ──────►  loadUserBindings.ts  ──────►  validate.ts
│  (built-in defaults)           (user ~/.claude/keybindings.json)      │
│                                    │                                    │
│                                    ▼                                    │
│                            ValidationErrors                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           TYPE DEFINITIONS                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  types.ts  ──────►  parser.ts  ──────►  reservedShortcuts.ts           │
│  (interfaces)        (parsing)           (forbidden keys)               │
│         │                 │                                               │
│         │                 ▼                                               │
│         │          resolver.ts                                            │
│         │          (core resolution logic)                               │
│         │                 │                                               │
│         └────────────────┬┘                                              │
│                          │                                               │
└──────────────────────────┼──────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           PROVIDER LAYER                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  KeybindingProviderSetup.tsx                                            │
│  ├── Loads & merges bindings (defaults + user overrides)               │
│  ├── Validates configuration                                            │
│  ├── Renders KeybindingContext.Provider                                 │
│  │       └── Resolver (resolver.ts)                                     │
│  │       └── Handler Registry (Map<action, Set<handlers>>)              │
│  │       └── Chord State Management                                     │
│  │       └── Active Context Tracking (Set<context>)                    │
│  └── ChordInterceptor (global useInput for chord sequences)              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         CONSUMER LAYER                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  useKeybinding.ts                                                       │
│  └── Registers action handlers with KeybindingContext                   │
│  └── useOptionalKeybindingContext() ◄──┐                                │
│                                         │                                │
│  useShortcutDisplay.ts                  │                                │
│  └── getDisplayText() ◄─────────────────┘                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

1. **Initialization**: `KeybindingProviderSetup.tsx` loads defaults from `defaultBindings.ts` and user config from `~/.claude/keybindings.json` via `loadUserBindings.ts`
2. **Validation**: `validate.ts` runs checks for parse errors, duplicates, reserved shortcuts, invalid contexts/actions
3. **Provider Creation**: `KeybindingContext.tsx` creates the context with `resolver.ts` logic, handler registry, chord state, and active context tracking
4. **Hot Reload**: File watcher monitors `keybindings.json` changes and re-validates/re-loads on modification
5. **Consumption**: Components use `useKeybinding()` to register action handlers; `useShortcutDisplay()` to get display text for UI labels

## Key Takeaways

| Area | Details |
|------|---------|
| **Chord Support** | Multi-key sequences (e.g., `ctrl+k ctrl+s`) are supported via `ChordInterceptor` using global `useInput` to capture keystrokes before other handlers; `CHORD_TIMEOUT_MS` (1000ms) defines sequence window |
| **Context Priority** | Active contexts (registered by focused components) take precedence over component's own context, which takes precedence over `Global` |
| **Platform Differences** | `defaultBindings.ts` handles Windows-specific differences: `alt+v` for image paste, `shift+tab` for mode cycling, VT mode support detection |
| **Feature Gating** | Bindings like `ctrl+shift+b` (KAIROS), `ctrl+shift+f` (QUICK_SEARCH), `meta+j` (TERMINAL_PANEL) are conditionally included via feature flags |
| **Unbindable Keys** | `ctrl+c` and `ctrl+d` are specially handled with double-press detection but cannot be rebound—validation in `reservedShortcuts.ts` enforces this |
| **Reserved Shortcuts** | A separate list prevents user overrides of OS/terminal reserved shortcuts |
| **Event Propagation** | `useKeybinding.ts` uses `stopImmediatePropagation()` to prevent conflicting handlers; returning `false` from a handler explicitly allows propagation |
| **Error Recovery** | `useShortcutDisplay.ts` has fallback strings with analytics tracking to safely handle migration away from fallback usage |
