# Summary of `skills/bundled/`

## Purpose of `skills/bundled/`

This directory implements the **bundled skills system** for the Claude Code CLI — a framework for registering and invoking self-contained, prompt-driven skills that require no external npm packages. Each skill ships as a TypeScript module with hardcoded prompts, documentation, and registration logic, all evaluated at startup.

The directory contains two categories of files:

| Category | Count | Description |
|----------|-------|-------------|
| **Skill Modules** | 8 | Files ending in `.ts` that implement individual skills (`askUser`, `simplify`, `claudeApi`, `updateConfig`, `verify`, `stuck`, `exitPlanMode`) plus the registry (`bundledSkills.ts`) |
| **Content Modules** | 2 | Files ending in `Content.ts` that inline large markdown blobs at build time (`claudeApiContent.ts`, `verifyContent.ts`) |
| **Entry Point** | 1 | `index.ts` re-exports the public registration API |

---

## Contents Overview

### Registry & Entry Point

| File | Role |
|------|------|
| `bundledSkills.ts` | Core registration system. Defines `BundledSkillSpec`, `SkillSpec`, and `BundledSkillMeta`. Exports `registerBundledSkill()` (used by all skills) and `getBundledSkills()` (consumed by the CLI at startup). Acts as middleware, enriching skill specs with metadata before forwarding to the CLI's skill registry. |
| `index.ts` | Thin re-export barrel that exposes `registerBundledSkill` and `getBundledSkills` under `bundledSkills/` paths for consumers in `src/`. |

### Individual Skills

| File | Skill Name | Purpose | Key Mechanism |
|------|-----------|---------|---------------|
| `askUser.ts` | `ask-user` | Pose questions to the user via the `AskUserQuestion` tool, presenting options as numbered choices with metadata. | Parses structured args (`--type`, `--question`, `--options`, `--default`) into tool calls; validates options/max choices. |
| `simplify.ts` | `simplify` | Invoke the simplify skill on the user's code. | Lazy-loads `simplifyContent.js` only when invoked; appends user request to base prompt. |
| `claudeApi.ts` | `claude-api` | Provide Claude API documentation, auto-detecting the user's language. | Detects language via `readdir`; dynamically imports `claudeApiContent.js` (247KB) lazily; builds prompt with language-filtered documentation. |
| `updateConfig.ts` | `update-config` | Edit `settings.json` (permissions, hooks, env vars, plugins). | Generates JSON Schema from Zod `SettingsSchema` to keep docs in sync; supports `hooks-only` mode. |
| `verify.ts` | `verify` | Run the application to verify a code change. | Parses frontmatter from `verifyContent.js`; supports optional user-provided arguments appended to prompt. |
| `stuck.ts` | `stuck` | Diagnose frozen/stuck Claude Code sessions on the machine. | Only loads for `USER_TYPE === 'ant'`; instructs Claude to run `ps` diagnostics and post results to Slack `#claude-code-feedback`. |
| `exitPlanMode.ts` | `exit-plan-mode` | Exit plan mode; consumed internally rather than user-invocable. | Sync function that returns an empty prompt; has no `getPromptForCommand`. |
| `skills.ts` | (internal) | Defines the internal `skills` bundled skill. | Loads skill definitions from `skillsContent.ts`; `userInvocable: false`, `disableModelInvocation: false` — consumed by the orchestrator. |

### Content Bundles

| File | Size | Content |
|------|------|---------|
| `claudeApiContent.ts` | ~247KB | Claude API documentation for Python, TypeScript, Go, Java, C#, Ruby, PHP, and curl — plus shared concepts (error codes, prompt caching, models, tool use). Bundled via Bun's text loader. |
| `verifyContent.ts` | small | `SKILL.md` + example files (`examples/cli.md`, `examples/server.md`) for the verify skill. |
| `simplifyContent.ts` | — | Prompt and files for the simplify skill (lazy-loaded). |
| `skillsContent.ts` | — | Skill definitions referenced by `skills.ts`. |

---

## How Files Relate to Each Other

```
index.ts (public API)
    │
    ├── bundledSkills.ts (registry core)
    │       │
    │       ├── registers all skills via registerBundledSkill():
    │       │
    │       ├── askUser.ts
    │       ├── simplify.ts ──────► simplifyContent.ts (lazy import)
    │       ├── claudeApi.ts ─────► claudeApiContent.ts (lazy import, 247KB)
    │       ├── updateConfig.ts ──► SettingsSchema (from utils/)
    │       ├── verify.ts ────────► verifyContent.ts (lazy import)
    │       ├── stuck.ts ─────────► (no content file, uses STUCK_PROMPT inline)
    │       ├── exitPlanMode.ts ──► (no content file, returns empty prompt)
    │       └── skills.ts ────────► skillsContent.ts (lazy import)
    │
    └── getBundledSkills() consumed by src/cli.ts at startup
```

**Dependency graph highlights:**

- `bundledSkills.ts` is the hub — all other files depend on it via `registerBundledSkill`.
- Content modules (`*Content.ts`) are **never imported at startup** — they are either:
  - **Lazy imports** inside `getPromptForCommand()` (e.g., `claudeApi.ts`, `verify.ts`), keeping startup fast.
  - **Inlined at build time** via Bun's text loader, so they carry no runtime I/O cost.
- `updateConfig.ts` stands apart — it imports from `../../utils/settings/types.js` (the Zod schema), using `toJSONSchema()` to generate documentation dynamically rather than bundling markdown.
- `claudeApi.ts` is the only skill that performs **runtime filesystem I/O** (`readdir` via `fs/promises`) to detect the user's programming language.

---

## Key Takeaways

1. **Zero external runtime dependencies.** Every skill is self-contained in a single `.ts` file. The largest content blob (247KB of API docs) is loaded lazily only when `/claude-api` is invoked.

2. **Two-stage registration pattern.** Skills declare themselves via `registerBundledSkill()` at module evaluation time. The CLI collects them via `getBundledSkills()` at startup. This decouples skill authors from the main CLI.

3. **Gating mechanisms.** Several skills are restricted:
   - `stuck.ts`, `verify.ts` — only register for `USER_TYPE === 'ant'`
   - `skills.ts` — `userInvocable: false`, consumed only by the orchestrator
   - `exitPlanMode.ts` — `userInvocable: false`, consumed only by the ExitPlanModeTool

4. **Dynamic prompt construction.** Most skills do non-trivial prompt building inside `getPromptForCommand()` (appending user args, filtering docs by language, injecting JSON schemas). This keeps the initial module load cheap.

5. **Build-time inlining.** Markdown files are compiled into `.js` bundles via Bun's text loader, eliminating runtime file reads for documentation — except for `claudeApi.ts`'s language detection, which genuinely reads the working directory.
