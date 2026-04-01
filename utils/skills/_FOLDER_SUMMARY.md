# Summary of `utils/skills/`

## Purpose of `skills/`

The `skills/` directory contains the core skill loading and management system for Claude Code. It handles discovering, loading, and caching custom skills from both user-level (`~/.claude/skills`) and project-level (`.claude/skills`) directories. The system is designed to detect changes to skill files in real-time and automatically invalidate caches to ensure the latest skill definitions are used.

## Contents Overview

| File | Role |
|------|------|
| `loadSkillsDir.js` | Handles loading skills from filesystem directories, manages skill caches (`clearSkillCaches`, `getSkillsPath`), and provides `onDynamicSkillsLoaded` callback |

The `skills/` directory works closely with `utils/skills/skillChangeDetector.ts`, which monitors the skill directories for file changes and triggers cache invalidation when modifications are detected.

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────┐
│                         skillChangeDetector.ts                   │
│  [File watcher - monitors .claude/skills & .claude/commands]      │
└───────────────────────────────┬──────────────────────────────────┘
                                │ imports
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                         loadSkillsDir.js                         │
│  [Skill loader - reads skill files, manages caches]              │
│  Exports: clearSkillCaches(), getSkillsPath(), onDynamicSkills   │
└──────────────────────────────────────────────────────────────────┘
```

When `skillChangeDetector.ts` detects a file change:
1. It calls `clearSkillCaches()` from `loadSkillsDir.js` to invalidate cached skills
2. It emits a signal that triggers reloading of skills from disk
3. The `onDynamicSkillsLoaded` callback is invoked to notify the system

## Key Takeaways

- **Dual-location support**: Skills can be defined at both user level (~/.claude/) and project level (.claude/)
- **Real-time updates**: File watching ensures skills are reloaded without restart
- **Cache management**: Aggressive caching for performance, with automatic invalidation on changes
- **Cross-platform polling**: Uses polling mode on Bun to avoid native watcher deadlocks during git operations
- **Debounced reloads**: Multiple rapid file changes are batched into a single reload event (300ms)
