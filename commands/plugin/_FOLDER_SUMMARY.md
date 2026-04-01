# Summary of `commands/plugin/`

## Purpose of `plugin/`

The `plugin/` directory contains the CLI plugin management system for **Claude Code**, built with **Ink** (React for terminal interfaces). It enables users to:

- Browse available plugins from multiple marketplaces
- Install, manage, and uninstall plugins
- Add custom marketplace sources
- Validate plugin manifest files
- Configure plugin trust and security policies

This is a fully terminal-based UI system — all rendering uses Ink components rather than web DOM.

## Contents Overview

| File | Purpose |
|------|---------|
| `BrowseMarketplace.tsx` | **Main orchestrator**: Multi-view navigation for marketplace browsing, plugin listing, details, and installation |
| `AddMarketplace.tsx` | Form component for adding custom marketplace sources (GitHub repos, URLs, local paths) |
| `UnifiedInstalledCell.tsx` | Row renderer displaying a single installed plugin or MCP server with status indicators |
| `PluginTrustWarning.tsx` | Component displaying trust warnings for plugins with customizable actions |
| `ValidatePlugin.tsx` | Plugin manifest validation runner with exit code management |
| `usePagination.ts` | Custom hook for virtual pagination/scrolling through large plugin lists |
| `PluginOptionsFlow.js` | Flow for handling plugin-specific configuration options during installation |
| `pluginDetailsHelpers.js` | Menu building, GitHub repo extraction, and `InstallablePlugin` type definitions |
| `unifiedTypes.js` | Type definitions for the unified plugin/MCP list view |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           User Commands                                   │
│                                                                           │
│   /plugin         → BrowseMarketplace.tsx (main entry point)             │
│   /plugin install → BrowseMarketplace.tsx (with targetPlugin)            │
│   /plugin add     → AddMarketplace.tsx                                   │
│   /plugin validate → ValidatePlugin.tsx                                 │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         BrowseMarketplace.tsx                             │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │  Views: marketplace-list → plugin-list → plugin-details        │    │
│   │                  ↓                              ↓               │    │
│   │         usePagination.ts              PluginOptionsFlow.js     │    │
│   └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                     │
│           ┌───────────────────────┼───────────────────────┐              │
│           ▼                       ▼                       ▼              │
│  UnifiedInstalledCell.tsx  PluginTrustWarning.tsx  pluginDetailsHelpers  │
│  (renders rows in lists)   (warns on install)         (menu building)     │
└──────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Hierarchy

1. **BrowseMarketplace** is the primary navigation controller
2. It uses **usePagination** for scrolling through large plugin lists
3. It delegates to **PluginOptionsFlow** when plugins require configuration
4. **PluginTrustWarning** provides security warnings before installation
5. **UnifiedInstalledCell** renders individual items (plugins or MCP servers)
6. **pluginDetailsHelpers** extracts GitHub URLs and builds menu structures
7. **ValidatePlugin** runs standalone validation without the browse UI

### Shared Utilities

| Utility | Location | Usage Across Components |
|---------|----------|------------------------|
| `isPluginInstalled()` | `installedPluginsManager.js` | BrowseMarketplace (counting), UnifiedInstalledCell (status) |
| `clearAllCaches()` | `cacheUtils.js` | BrowseMarketplace, AddMarketplace |
| `logEvent()` | `analytics` | AddMarketplace (tracking adds) |
| `isPluginBlockedByPolicy()` | `pluginPolicy.js` | BrowseMarketplace (filtering) |
| `getInstallCounts()` | `installCounts.js` | BrowseMarketplace (popularity sorting) |

## Key Takeaways

### Architecture Patterns

1. **Multi-view navigation**: BrowseMarketplace manages a `ViewState` union type (`'marketplace-list' | 'plugin-list' | 'plugin-details' | {type: 'plugin-options', ...}`) that controls what renders and which keyboard shortcuts apply.

2. **Ink CLI rendering**: All UI components use Ink's `<Box>` and `<Text>` primitives with theme colors (`"claude"`, `"suggestion"`, `"error"`, `"warning"`, `"success"`, `"inactive"`).

3. **Dual-mode execution**: Components support both interactive CLI mode (navigates views) and non-interactive CLI mode (sets `result` string for scriptable use).

4. **Graceful degradation**: Marketplace loading failures are non-fatal — BrowseMarketplace shows warnings but continues with available marketplaces.

### Security & Trust

- **Policy-based blocking**: Plugins can be blocked by organizational policy via `isPluginBlockedByPolicy()`
- **Trust warnings**: `PluginTrustWarning` allows per-plugin trust decisions before installation
- **Validation**: `ValidatePlugin` checks manifest JSON schema before installation

### Performance Optimizations

- **usePagination hook**: Virtual scrolling keeps only visible items in memory
- **React Compiler**: Most components use `_c` from `react/compiler-runtime` for automatic memoization
- **Ref guards**: `AddMarketplace` uses `useRef` to prevent duplicate auto-add calls

### Cache Management

After plugin installations or marketplace additions, `clearAllCaches()` invalidates:
- Loaded marketplace data
- Installed plugin metadata
- Install count statistics

This ensures subsequent operations reflect the latest state.

### Analytics Integration

`AddMarketplace` tracks `tengu_marketplace_added` events with `source_type` metadata for telemetry.
