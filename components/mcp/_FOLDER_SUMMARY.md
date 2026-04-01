# Summary of `components/mcp/`

## Purpose of `components/mcp/`

The `components/mcp/` directory contains React components (built with **Ink**, a React renderer for CLI/Terminal) that implement the **Model Context Protocol (MCP) UI layer**. This includes server management interfaces, tool browsing and selection dialogs, and elicitation workflows for requesting user input during AI tool execution.

## Contents Overview

| File | Description |
|------|-------------|
| `CapabilitiesSection.tsx` | Displays MCP server capability counts (tools, resources, prompts) |
| `ElicitationDialog.tsx` | ~1200-line complex dialog for eliciting user input via forms or URL; supports text, enum, multi-select, and date/datetime fields |
| `MCPAgentServerMenu.tsx` | Interactive menu for managing agent-type MCP servers |
| `MCPListPanel.tsx` | Main panel displaying all MCP servers with status indicators and quick actions |
| `MCPReconnect.tsx` | Reconnection UI component with spinner and fallback options |
| `MCPRemoteServerMenu.tsx` | Menu interface for remote MCP servers |
| `MCPSettings.tsx` | Settings/configuration view for MCP servers |
| `MCPStdioServerMenu.tsx` | Menu for stdio-based MCP servers with configuration display |
| `MCPToolDetailView.tsx` | Detail view showing tool metadata, parameters, and flags |
| `MCPToolListView.tsx` | Selectable list of server tools with annotations |
| `index.ts` | Barrel export for all components and types |
| `types.ts` | Shared TypeScript type definitions |
| `utils/` subdirectory | Helper functions (e.g., `reconnectHelpers.ts`) |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Entry Points                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  MCPListPanel                    MCPSettings                            │
│  (Server list overview)          (Configuration view)                   │
└───────────────┬─────────────────────────────────┬───────────────────────┘
                │                                 │
                ▼                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        Server Menu Components                              │
├───────────────────────────────────────────────────────────────────────────┤
│  MCPAgentServerMenu         MCPRemoteServerMenu        MCPStdioServerMenu │
│         │                        │                        │               │
│         └────────────┬───────────┘                        │               │
│                      ▼                                    ▼               │
│              MCPReconnect                           CapabilitiesSection   │
│              (Spinner + fallback)                   (counts display)     │
└───────────────────────────────────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                          Tool Browsing                                     │
├───────────────────────────────────────────────────────────────────────────┤
│  MCPToolListView ──────────────► MCPToolDetailView                         │
│  (Selectable tool list)          (Tool parameters, flags, description)    │
└───────────────────────────────────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      Elicitation Workflow                                  │
├───────────────────────────────────────────────────────────────────────────┤
│  ElicitationDialog                                                         │
│  ├── ElicitationURLDialog (OAuth/URL-based elicitation)                    │
│  └── ElicitationFormDialog (Form-based: text, enum, date, multi-select)   │
└───────────────────────────────────────────────────────────────────────────┘
```

**Dependency graph:**
- All components depend on `ink.js` for terminal UI primitives (`Box`, `Text`, `useInput`, etc.)
- Tool views (`MCPToolListView`, `MCPToolDetailView`) depend on `CapabilitiesSection` for capability counts
- `MCPToolListView` imports `MCPToolDetailView`-related utilities (`extractMcpToolDisplayName`, `getMcpDisplayName`)
- `MCPSettings` imports all server menu components
- `MCPStdioServerMenu` imports `MCPReconnect` and `CapabilitiesSection`
- All components use shared design system components (`Dialog`, `Byline`, `ConfigurableShortcutHint`)
- Type definitions flow from `types.ts` and `Tool.js` throughout

## Key Takeaways

### Architecture Pattern
- **Master-detail UI pattern**: `MCPToolListView` → `MCPToolDetailView` mirrors the server list → server menu pattern
- **Barrel export (`index.ts`)**: Clean public API for the entire MCP UI module
- **Component composition**: Complex dialogs (`ElicitationDialog`) wrap simpler presentational components

### React Compiler Usage
All components are pre-compiled with **React Compiler (Babel transform)**, evidenced by:
- `react/compiler-runtime` imports with `_c` function usage
- Memoization via `$[]` cache arrays (8–44 slots depending on component complexity)
- Pattern: `if ($[n] !== value) { $[n] = computed; } return $[n];` for stale checks

### Ink UI Framework
- All components render **terminal-compatible UI** using Ink's `Box`, `Text`, and hooks
- No DOM dependencies — pure terminal output
- Supports keyboard navigation (`useInput`, `useKeybinding`)

### State Management
- Global state via `useAppState()` hook (reads from `mcp` slice)
- Local state via `useState()` for transient UI state (form values, focused fields)
- Keyboard shortcuts via `useKeybinding()` and `useExitOnCtrlCDWithKeybindings()`

### Feature Highlights
- **ElicitationDialog**: Most complex component — supports async field resolution, typeahead for enums, date/datetime parsing, multi-select validation
- **CapabilitiesSection**: Lightweight utility component that filters non-zero capability counts
- **MCPReconnect**: Isolated spinner component (80ms animation) to prevent full form re-renders
- **Tool annotations**: Visual indicators for `read-only`, `destructive`, and `open-world` tool flags

### Shared Utilities
- `services/mcp/utils.js` — Server filtering, config path formatting, string utilities
- `services/mcp/MCPConnectionManager.js` — Connection state management hooks
- `components/mcp/utils/reconnectHelpers.ts` — Result/error formatting for reconnection flows
- Design system components for consistent styling across the MCP UI
