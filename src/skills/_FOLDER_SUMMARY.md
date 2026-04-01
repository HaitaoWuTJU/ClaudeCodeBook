# Summary of `skills/`

## Purpose of `skills/`

This directory is the central hub for **skill management** in the Claude Code CLI — handling both **built-in (bundled) skills** that ship compiled into the binary and **disk-based skills** loaded from various config directories. It provides the registration APIs, loading logic, deduplication, dynamic discovery, and MCP (Model Context Protocol) integration that power the entire skill system.

## Contents Overview

| File / Directory | Role |
|------------------|------|
| `index.ts` | Public-facing barrel export — re-exports `registerBundledSkill`, `getBundledSkills`, `getCommandDirCommands`, `clearSkillCaches`, and dynamic skill APIs for consumers in `src/`. |
| `bundledSkills.ts` | Core registry for bundled skills. Defines `BundledSkillSpec` and `SkillSpec` types; exports `registerBundledSkill()` for skill registration and `getBundledSkills()` for collection. Acts as middleware, enriching specs with metadata (symlink protection, memoized extraction promises, base directory prepending). |
| `loadSkillsDir.ts` | Loads **disk-based skills** from multiple sources (managed, user, project, plugin, additional `--add-dir` paths). Handles legacy `/commands/` directories, modern `/skills/` directories, deduplication via `realpath`, conditional path-based activation, inline shell command execution, and dynamic skill discovery during file operations. |
| `mcpSkillBuilders.ts` | A **write-once registry** that solves a circular dependency: `loadSkillsDir.ts` provides `createSkillCommand` and `parseSkillFrontmatterFields`, but `mcpSkills.ts` needs them without creating a cycle. Registration happens eagerly at module load time; retrieval throws if accessed prematurely. |
| `mcpSkills.ts` | MCP server implementation for dynamic skill discovery. Connects skill builders (from `mcpSkillBuilders`) to the MCP protocol, enabling remote MCP servers to expose skills that behave identically to disk-based ones. |
| `bundled/` | Subdirectory containing all **built-in skill implementations**. Each skill is a self-contained TypeScript module with hardcoded prompts and optional markdown content (bundled via Bun's text loader). The `index.ts` inside re-exports the registry API. |

## How Files Relate to Each Other

```
CLI startup (src/cli.ts)
    │
    ├─────────────────────────────────────────────────────────────┐
    │ skills/index.ts (barrel)                                   │
    │       │                                                     │
    │       ├──► bundledSkills.ts ◄──► bundled/*.ts             │
    │       │       (registerBundledSkill, getBundledSkills)     │
    │       │                                                     │
    │       └──► loadSkillsDir.ts                                │
    │               │                                             │
    │               ├──► mcpSkillBuilders.ts (write-once store)  │
    │               │         ▲                                   │
    │               │         │ registers at module eval time    │
    │               │         │                                   │
    │               └──► mcpSkills.ts                            │
    │                           │                                 │
    │                           ▼                                 │
    │                   MCP protocol (dynamic discovery)          │
    └─────────────────────────────────────────────────────────────┘

Disk-based skills (loaded at startup)
    sources: managed (policy) → user (~/.config) → project (.claude/) → plugin → additional dirs

Bundled skills (registered eagerly at startup)
    sources: bundled/*.ts (compiled into binary)
    loading: lazy for large content (claudeApiContent.ts, verifyContent.ts, simplifyContent.ts)

Dynamic skills (discovered during session)
    triggered by: file operations, onDynamicSkillsLoaded() callbacks
    filters: gitignored dirs skipped, conditional skills activated on path match
```

**Dependency cycle prevention:**
`loadSkillsDir.ts` imports only *types* from `mcpSkillBuilders.ts`, then calls `registerMCPSkillBuilders()` at module evaluation time. `mcpSkills.ts` calls `getMCPSkillBuilders()` to retrieve the builders. A direct static import in either direction would create a cycle.

**Bundled vs. disk-based loading paths:**
- **Bundled**: `bundled/index.ts` → `bundled/bundledSkills.ts` → individual `bundled/*.ts` → content `bundled/*Content.ts`
- **Disk-based**: `loadSkillsDir.ts` → `loadSkillsFromSkillsDir()` / `loadSkillsFromCommandsDir()` → per-skill `SKILL.md` / `skill.md`
- Both paths converge at `createSkillCommand()`, which produces a `Command` object with `getPromptForCommand()`.

## Key Takeaways

1. **Two distinct skill systems, one unified interface.** Bundled skills (compiled into the binary) and disk-based skills (loaded from config directories) both produce `Command` objects with `getPromptForCommand()`. The CLI consumes them identically.

2. **Multi-source loading with first-wins deduplication.** Skills are loaded in parallel from managed, user, project, plugin, and additional directories. `realpath` resolves symlinks to prevent the same skill from loading twice via different paths. The first source to provide a given file identity wins.

3. **Lazy content loading keeps startup fast.** The largest bundled content (247KB of API documentation in `claudeApiContent.ts`) is imported only inside `getPromptForCommand()`, not at module evaluation time. This keeps CLI startup fast despite the large blobs.

4. **Dynamic discovery bridges static loading and runtime needs.** Skills discovered in `.claude/skills/` directories near the user's working files are loaded lazily when matching paths are touched. Callbacks via `onDynamicSkillsLoaded()` allow other systems (e.g., MCP servers) to react.

5. **Conditional skills enable path-gated activation.** Skills can declare `paths` frontmatter (gitignore-style patterns). They remain dormant until a file matching one of those patterns is created or modified, then activate via `activateConditionalSkillsForPaths()`.

6. **Variable substitution and shell execution extend skill power.** Skill content supports:
   - `${CLAUDE_SKILL_DIR}` — the skill's directory for `Read`/`Grep` commands
   - `${CLAUDE_SESSION_ID}` — current session identifier
   - Inline `!bash` blocks — executed via `executeShellCommandsInPrompt` for local skills
   - MCP skills skip shell execution (content is remote/untrusted)

7. **Security hardening throughout.** Bundled skill file extraction uses `O_NOFOLLOW | O_EXCL` flags and `0o600`/`0o700` permissions to prevent symlink attacks. Dynamic discovery skips gitignored directories. Path traversal (`..`) and absolute paths are rejected in skill file paths.

8. **Gated skills enforce internal-only usage.** Some skills (`stuck`, `verify`, `exit-plan-mode`, `skills`) have `userInvocable: false` or are restricted to `USER_TYPE === 'ant'`, ensuring they are consumed only by internal tooling rather than exposed to end users.
