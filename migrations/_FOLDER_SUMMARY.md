# Summary of `migrations/`

## Purpose of `migrations/`

The `migrations/` directory houses **one-shot, idempotent migration scripts** that safely transition user configuration from an older schema or location to a newer one. These migrations run at application startup (or on demand) to ensure that existing users' settings are brought up to date without requiring manual intervention. Each file exports a single `function` that is called once, self-validates its preconditions, and marks completion in either `globalConfig` or a settings file to prevent re-execution.

---

## Contents Overview

The directory contains **8 migration files** that fall into three logical groups:

### Group 1 — Settings Relocation (3 files)

These migrations move user preference fields from `globalConfig` into `settings.json`, reflecting a deliberate architectural decision to keep user-scoped preferences in a dedicated settings layer.

| File | From | To |
|---|---|---|
| `migrateAutoUpdatesToSettings.ts` | `globalConfig.autoUpdates`, `autoUpdatesProtectedForNative` | `settings.json` → `env.DISABLE_AUTOUPDATER: '1'` |
| `migrateBypassPermissionsAcceptedToSettings.ts` | `globalConfig.bypassPermissionsModeAccepted` | `settings.json` → `skipDangerousModePermissionPrompt` |
| `migrateEnableAllProjectMcpServersToSettings.ts` | `globalConfig.enableAllProjectMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers` | `localSettings` (deduplicates server lists with `Set`) |

### Group 2 — Model Alias Upgrades (4 files)

These migrations update pinned model identifier strings to their modern equivalents, progressively advancing users through a chain of model aliases. All four are **mutually exclusive per-user** (guarded by completion flags in `globalConfig`).

| File | Migration | Target audience |
|---|---|---|
| `migrateFennecToOpus.ts` | `fennec-latest` → `opus`, `fennec-fast-latest` → `opus[1m]` + `fastMode: true` | First-party (`ant`) users with deprecated Fennec aliases |
| `migrateSonnet1mToSonnet45.ts` | `sonnet[1m]` → `sonnet-4-5-20250929[1m]` | Users who had the `sonnet[1m]` alias pinned |
| `migrateSonnet45ToSonnet46.ts` | `claude-sonnet-4-5-*` / `sonnet-*` → `sonnet` (alias) | First-party Pro/Max/Team Premium on explicit Sonnet 4.5 strings |
| `resetProToOpusDefault.ts` | Marks migration complete for first-party Pro users (no model change; timestamp recorded) | First-party Pro subscribers without custom model settings |

### Group 3 — Dialog / Permission State Reset (1 file)

| File | Purpose |
|---|---|
| `resetAutoModeOptInForDefaultOffer.ts` | Re-surfaces the AutoModeOptInDialog for ~40 users who accepted the old 2-option dialog but don't have `'auto'` as their default mode, so they can see the new "make it my default" option |

---

## How Files Relate to Each Other

```
Configuration Storage Hierarchy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
globalConfig (~/.claude.json)
  ├── autoUpdates                          ← migrateAutoUpdatesToSettings
  ├── autoUpdatesProtectedForNative        ← migrateAutoUpdatesToSettings
  ├── bypassPermissionsModeAccepted       ← migrateBypassPermissionsAcceptedToSettings
  ├── enableAllProjectMcpServers           ← migrateEnableAllProjectMcpServersToSettings
  ├── enabledMcpjsonServers                ← migrateEnableAllProjectMcpServersToSettings
  ├── disabledMcpjsonServers               ← migrateEnableAllProjectMcpServersToSettings
  ├── sonnet1m45MigrationComplete          ← migrateSonnet1mToSonnet45
  ├── sonnet45To46MigrationTimestamp       ← migrateSonnet45ToSonnet46
  ├── opusProMigrationComplete             ← resetProToOpusDefault
  ├── opusProMigrationTimestamp            ← resetProToOpusDefault
  ├── hasResetAutoModeOptInForDefaultOffer ← resetAutoModeOptInForDefaultOffer
  └── numStartups                          ← shared read (version-gating)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
settings.json (userSettings)
  ├── env.DISABLE_AUTOUPDATER: '1'         ← migrateAutoUpdatesToSettings
  ├── skipDangerousModePermissionPrompt     ← migrateBypassPermissionsAcceptedToSettings
  └── model: 'sonnet' | 'opus' | ...       ← migrateFennecToOpus, migrateSonnet1mToSonnet45, migrateSonnet45ToSonnet46
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
localSettings
  └── enableAllProjectMcpServers / enabledMcpjsonServers / disabledMcpjsonServers ← migrateEnableAllProjectMcpServersToSettings
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Cross-file relationships:**

- **Model chain**: `migrateFennecToOpus` → `migrateSonnet1mToSonnet45` → `migrateSonnet45ToSonnet46` → `resetProToOpusDefault`. These are intentionally ordered so each migration moves users one step forward in the model release chain. The first three each own a distinct completion flag; `resetProToOpusDefault` is a catch-all that marks Pro-to-Opus complete regardless of starting point.
- **Settings vs. globalConfig**: The three relocation migrations write to `settings.json`/`localSettings` but clean up only the old keys in `globalConfig`. They do **not** delete the entire `globalConfig` entry — they use object rest spread to surgically remove only the migrated fields.
- **Feature flag cascade**: `resetAutoModeOptInForDefaultOffer` is gated by `TRANSCRIPT_CLASSIFIER`, meaning its migration only runs when that experimental feature is enabled.
- **Broadcast of runtime state**: `migrateAutoUpdatesToSettings` is unique in that it writes both to `settings.json` **and** to `process.env.DISABLE_AUTOUPDATER` at runtime, so the effect is immediate without a restart.

---

## Key Takeaways

1. **All migrations are idempotent.** Every file checks for a completion flag (`*MigrationComplete`, `*Timestamp`, or `hasReset*`) before taking action, ensuring that running the migration multiple times is safe.

2. **GlobalConfig is the canonical registry of "already done."** Completion flags live in `globalConfig` (not in settings) so they survive settings resets and are shared across all entry points.

3. **Settings-layer migrations do targeted cleanup only.** Instead of wholesale deletion of old config keys, each migration uses object rest syntax (`{ oldKey: _, ...rest }`) to surgically remove only the fields that were migrated.

4. **Analytics events are first-class citizens.** Every migration emits at least one `logEvent` call, tagging outcomes with `skipped`, `has_*`, `from_model`, and other contextual flags. This makes it possible to measure migration reach and reversion rates post-launch.

5. **Failures are swallowed, not thrown.** Every migration wraps its core logic in a `try-catch` that logs the error but does **not** re-throw. This prevents a failed migration from crashing the application at startup — a deliberate design choice for production resilience.

6. **Scoped reads vs. merged reads:** Model migrations read from `getSettingsForSource('userSettings')` directly (not the merged settings layer) so that project-local or policy-level overrides are left untouched. This prevents a project-scoped model pin from being promoted to the global default during migration.

7. **Environment-type gating**: `migrateFennecToOpus` runs only when `process.env.USER_TYPE === 'ant'`, and `resetAutoModeOptInForDefaultOffer` is gated by the `TRANSCRIPT_CLASSIFIER` feature flag — demonstrating a consistent pattern of using environment variables and feature flags as migration gates.
