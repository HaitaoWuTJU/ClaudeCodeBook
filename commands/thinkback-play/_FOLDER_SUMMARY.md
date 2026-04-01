# Summary of `commands/thinkback-play/`

## Purpose of `thinkback-play/`

Implements the `/thinkback-play` command, a hidden internal command that plays the thinkback animation skill. It validates that the thinkback plugin is installed before triggering the animation, with marketplace resolution based on user type.

## Contents Overview

| File | Description |
|------|-------------|
| `thinkback-play.ts` | Single-file command implementation (~1.4KB) |

## How Files Relate to Each Other

```
thinkback-play.ts
    │
    ├── imports → commands.js (LocalCommandResult type)
    ├── imports → installedPluginsManager.js (loadInstalledPluginsV2)
    ├── imports → officialMarketplace.js (OFFICIAL_MARKETPLACE_NAME)
    └── imports → ../thinkback/thinkback.js (playAnimation)
```

**Data flow within file:**

```
getPluginId()                    ← Reads USER_TYPE env var
       ↓
loadInstalledPluginsV2()         ← Fetches plugin registry
       ↓
Filter installations by pluginId ← Validates plugin exists
       ↓
Extract installPath             ← Gets plugin directory
       ↓
Build skillDir path             ← Resolves skills/thinkback
       ↓
playAnimation(skillDir)          ← Executes the animation
       ↓
Return LocalCommandResult       ← Reports success/failure
```

## Key Takeaways

1. **Marketplace-aware**: Uses internal marketplace (`claude-code-marketplace`) for `ant` users, official marketplace for everyone else

2. **Fail-safe**: Gracefully handles missing plugin with user-friendly error message rather than crashing

3. **Lazy-loaded**: The actual animation logic (`thinkback.js`) is only imported when the command executes

4. **Single installation assumption**: Accesses `installations[0]` assuming only one installation per user

5. **Hidden command**: Not exposed in public command listings; intended for internal use by the thinkback skill
