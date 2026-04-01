# Summary of `commands/reload-plugins/`

## Purpose of `reload-plugins/`

Provides the `/reload-plugins` command implementation that activates pending plugin changes in the running session. It handles the full refresh cycle: optionally syncing user settings from remote, notifying the change detector, and reloading all active plugins, skills, agents, hooks, MCP servers, and LSP servers.

## Contents Overview

| File | Role | Public API |
|------|------|------------|
| `index.ts` | Command registration entry point | Exports `reloadPlugins` default satisfying `Command` interface |
| `reload-plugins.ts` | Core implementation | Exports `call` handler and `n()` helper |

## How Files Relate to Each Other

```
commands/
└── reload-plugins/
    ├── index.ts          ← Lazy-loaded entry (satisfies Command interface)
    │   load: () => import('./reload-plugins.js')
    │   │
    │   └── reload-plugins.ts  ← Actual handler
    │       ├── call()         ← LocalCommandCall handler
    │       ├── refreshActivePlugins()  ← Core refresh logic
    │       └── n()            ← Pluralization helper
```

**Runtime flow**:
1. Commands registry imports `index.ts` → gets metadata + loader function
2. On `/reload-plugins` invocation → `load()` executes dynamic import
3. `reload-plugins.ts`'s `call()` runs, interacting with:
   - `redownloadUserSettings()` (remote mode only)
   - `settingsChangeDetector` (notifies mid-session change handlers)
   - `refreshActivePlugins()` (actual plugin/skill/agent/hook/MCP/LSP refresh)

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Lazy Loading** | Implementation is code-split; only loaded when command is first invoked |
| **Remote Awareness** | User settings re-download only in remote mode + when `DOWNLOAD_USER_SETTINGS` flag enabled |
| **Mid-session Refresh** | Uses `notifyChange()` to trigger `applySettingsChange` handlers (vs startup `markInternalWrite`) |
| **Fail-Open** | No retries on settings sync; single attempt with error included in response |
| **Non-Interactive Blocked** | `supportsNonInteractive: false` prevents execution in headless/CI environments |
| **Return Format** | `{ type: 'text', value: string }` with dot-separated item counts and optional error |
