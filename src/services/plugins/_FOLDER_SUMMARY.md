# Summary of `services/plugins/`

## Purpose of `services/plugins/`

The `plugins/` directory contains the core plugin lifecycle management system for what appears to be a Claude CLI tool. It handles plugin operations (install, uninstall, enable, disable, update) through a layered architecture that separates concerns between pure operations, CLI integration, and background startup reconciliation.

## Contents Overview

| File | Role | Key Responsibility |
|------|------|-------------------|
| **`pluginOperations.ts`** | Core logic layer | Pure library functions for plugin lifecycle: `installPluginOp`, `uninstallPluginOp`, `enablePluginOp`, `disablePluginOp`, `updatePluginOp`. Handles settings-first writes, policy enforcement, reverse dependency analysis, and V1/V2 installation records. |
| **`PluginInstallationManager.ts`** | Background reconciler | Manages non-blocking startup reconciliation of marketplace installations. Computes marketplace diffs, auto-installs missing plugins, triggers plugin refresh on new installs, and signals manual refresh for updates. |
| **`pluginCliCommands.ts`** | CLI adapter layer | Wraps `pluginOperations.ts` functions with CLI concerns: console output formatting (✓/✗ symbols), process exit codes, and analytics/telemetry event emission. |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────────┐
│                         STARTUP / CLI ENTRY                          │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PluginInstallationManager.ts                                       │
│  performBackgroundPluginInstallations()                              │
│  ├── diffMarketplaces() → identify missing/changed marketplaces     │
│  └── reconcileMarketplaces() → call installPluginOp internally      │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│  pluginOperations.ts                                                 │
│  Core Operations:                                                    │
│  ├── installPluginOp() → settings-first: search → write → cache     │
│  ├── uninstallPluginOp() → settings update + cache cleanup           │
│  ├── enablePluginOp() / disablePluginOp() → settings writes         │
│  ├── disableAllPluginsOp() → aggregated batch operation             │
│  └── updatePluginOp() → non-inplace: download → versioned cache     │
│                                                                          │
│  Dependencies:                                                        │
│  ├── Settings API (settings.js) — source of truth                    │
│  ├── Marketplace Manager — plugin discovery                           │
│  ├── Installed Plugins Manager — V1/V2 records                       │
│  ├── Plugin Loader — cache management                                │
│  └── Policy/Dependency/Versioning utilities                          │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│  pluginCliCommands.ts                                                │
│  CLI Wrappers:                                                        │
│  ├── installPlugin() → wraps installPluginOp + console output         │
│  ├── uninstallPlugin() → wraps uninstallPluginOp + console output    │
│  ├── enablePlugin() / disablePlugin() / disableAllPlugins()         │
│  └── updatePluginCli() → wraps updatePluginOp + gracefulShutdown     │
│                                                                          │
│  Adds:                                                                │
│  ├── Analytics events (tengu_plugin_installed_cli, etc.)            │
│  ├── Telemetry fields (with PII-tagged _PROTO_* variants)           │
│  └── Error classification for consistent analytics                    │
└──────────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Settings   │ ←── │ pluginOperations │ ←── │  CLI commands    │
│  (source    │     │ (core logic)     │     │  (user input)    │
│  of truth)  │ ──→ └──────────────────┘ ──→ └──────────────────┘
└─────────────┘              │                       │
                              ▼                       ▼
                    ┌──────────────────┐     ┌──────────────────┐
                    │  Plugin Cache    │     │  Console Output  │
                    │  (materialized   │     │  + Exit Codes    │
                    │   state)         │     │  + Analytics     │
                    └──────────────────┘     └──────────────────┘
```

## Key Takeaways

1. **Settings-first architecture**: All plugin state changes (install/enable/disable) write to settings files as the source of truth. Plugin cache materialization is a separate concern handled by the reconciler at startup.

2. **Non-blocking startup**: `PluginInstallationManager` handles marketplace cloning and plugin caching in the background without blocking application startup. It updates UI state (`installationStatus`) via progress callbacks.

3. **Idempotent operations**: Enable/disable operations check current state before writing, avoiding redundant settings modifications. Supports cross-scope overrides (e.g., local `false` overrides project `true`).

4. **Non-inplace updates**: Plugin updates download to a new versioned cache directory rather than modifying the existing installation, enabling rollback and clean cleanup of orphaned versions.

5. **Comprehensive telemetry**: All CLI commands emit analytics events (`tengu_plugin_*_cli`) with PII-safe (`_PROTO_*`) field variants for plugin/marketplace names.

6. **Multi-scope support**: Plugins can be installed at user, project, or local scopes with clear precedence (local > project > user) and appropriate visibility controls.

7. **Policy and dependency guards**: Install/enable operations check against plugin policies and warn about reverse dependencies to prevent breaking dependent plugins.

8. **Graceful degradation**: When auto-refresh fails after new installations, the system sets `needsRefresh = true` rather than failing outright, allowing manual intervention via `/reload-plugins`.
