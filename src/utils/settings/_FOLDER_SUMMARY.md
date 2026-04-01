# Summary of `utils/settings/`

## Purpose of `settings/`

The `utils/settings/` directory provides a comprehensive settings management system for Claude Code. Its responsibilities include:

- **Loading** settings from user, project, local, and policy (MDM) sources
- **Validating** settings against a Zod schema with user-friendly error messages
- **Watching** for file changes and propagating updates to listeners
- **Enforcing** MDM/enterprise policy overrides with first-source-wins semantics
- **Breaking circular dependencies** between settings, MCP config, and hooks modules

## Contents Overview

| File | Purpose |
|------|---------|
| `constants.ts` | Shared constants (setting sources, permission modes, effort levels, file paths) |
| `settings.ts` | Core loading with caching, internal write detection, cache invalidation |
| `validation.ts` | Validates JSON content against schema, transforms Zod errors to structured errors |
| `validationTips.ts` | Maps validation errors to user-friendly suggestions and doc links |
| `validateEditTool.ts` | Pre/post-edit validation for FileEditTool integration |
| `applySettingsChange.ts` | Applies changes to AppState: reloads settings, syncs permissions, updates hooks |
| `changeDetector.ts` | Filesystem watcher (chokidar) + MDM polling, emits change signals |
| `internalWrites.ts` | Tracks internal writes to prevent circular change notifications |
| `managedPath.ts` | Managed-settings.d drop-in directory path resolution |
| `schemaOutput.ts` | Generates JSON Schema from Zod schema for error context |
| `index.ts` | Combines settings + MCP errors, resolves circular imports |
| `mdm/` | Cross-platform MDM integration (plist/registry/JSON) |

## How Files Relate to Each Other

### Initialization Sequence

```
CLI/app startup
    │
    ├─► settings/constants.ts ──── Loaded first (pure config, no side effects)
    │
    ├─► changeDetector.ts
    │     ├─► initialize() ──── Sets up chokidar watcher
    │     └─► startMdmPoll() ──── Starts 30-min MDM registry polling
    │
    ├─► mdm/settings.ts ────────── Pre-fetches MDM via mdm/rawRead.ts
    │
    └─► settings/settings.ts ───── Loads initial settings
          ├─► getInitialSettings() ──► readSettingsFile() + mergeSettings()
          └─► applySettingsChange() ──► Applies to AppState
```

### Change Detection → Settings Update Flow

```
User/Policy/Script edits settings file
    │
    ▼
changeDetector.ts (chokidar watcher)
    │
    ├─► handleChange() / handleAdd() / handleDelete()
    │     ├─► consumeInternalWrite() ── Skip if internal
    │     ├─► executeConfigChangeHooks()
    │     └─► fanOut() ──── Single cache reset + signal emission
    │
    ▼
settingsChanged.emit(source)
    │
    ├─► Subscribers in AppState.tsx, print.ts, etc.
    │     └─► applySettingsChange() ──► Reloads from disk, updates state
    │
    └─► Subscribers in changeDetector.ts
          └─► startMdmRawRead() ── Re-fetches MDM if policy changed
```

### Validation Pipeline

```
FileEditTool (user edits file)
    │
    ▼
validateEditTool.ts ──► isClaudeSettingsPath() check
    │
    ▼
validation.ts
    ├─► validateSettingsFileContent()
    │     ├─► jsonParse(content)
    │     ├─► SettingsSchema.strict().safeParse()
    │     └─► formatZodError() ──► ValidationError[] with paths/messages
    │
    └─► filterInvalidPermissionRules() ── Pre-filters bad rules
              │
              ▼
        validationTips.ts ──► getValidationTip() ──► Suggestion + docLink
              │
              ▼
        User sees: error message + "Valid values: X, Y, Z" + documentation link
```

### Key Dependency Graph

```
                      ┌─────────────────────┐
                      │  settings/constants │  (no dependencies)
                      └──────────┬──────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
   ┌──────────────┐    ┌───────────────┐    ┌──────────────────┐
   │ settings.ts  │    │ changeDetector │    │   mdm/settings   │
   └───────┬──────┘    └───────┬───────┘    └────────┬─────────┘
           │                   │                    │
           │                   ▼                    ▼
           │          ┌────────────────┐    ┌───────────────┐
           │          │  internalWrites │    │  mdm/rawRead  │
           │          └────────────────┘    └───────┬───────┘
           │                                     │
           ▼                                     ▼
   ┌──────────────┐                       ┌─────────────┐
   │ validation.ts│◄──────────────────────│mdm/constants│
   └───────┬──────┘                       └─────────────┘
           │
           ▼
   ┌──────────────┐
   │validationTips│
   └──────────────┘
```

## Key Architectural Decisions

### 1. **Single Cache + Fan-Out Pattern** (changeDetector.ts → settings.ts)
Instead of each subscriber invalidating the cache independently (causing N redundant disk reads for N listeners), the cache is reset exactly once in `fanOut()` before all subscribers are notified.

### 2. **Internal Write Detection** (internalWrites.ts)
Internal writes (e.g., policy overrides, managed-settings updates) are tracked and consumed by `changeDetector.ts` to prevent circular change → reload → change cycles.

### 3. **MDM Pre-Fetch at Module Evaluation** (mdm/settings.ts)
`startMdmRawRead()` fires subprocess spawns synchronously during module load, before the event loop polls. This prevents blocking while ensuring MDM settings are available before initial settings merging.

### 4. **Circular Dependency Resolution** (index.ts)
`index.ts` imports both `settings.ts` and `mcp/config.ts` without being imported by either, breaking the import cycle between those two modules.

### 5. **Lenient Edit Validation** (validateEditTool.ts)
Editing invalid current files is allowed; only the *result* of the edit must be valid. This prevents users from being locked out of fixing corrupted settings.

### 6. **Permission Rule Resilience** (validation.ts)
Invalid individual permission rules are filtered out rather than rejecting the entire file. This allows MDM policies to push partial valid configurations.

### 7. **Zod v4 Compatibility**
Multiple files use type guards (`isInvalidTypeIssue`, etc.) because Zod v4 changed error structure from v3. The `typeof ZodIssueCode[keyof typeof ZodIssueCode]` pattern is used for type-safe enum access.

### 8. **First-Source-Wins for MDM**
Enterprise settings from multiple sources (plist, HKLM, managed-settings.d) are merged with clear priority hierarchy, preventing conflicting policies from being applied simultaneously.
