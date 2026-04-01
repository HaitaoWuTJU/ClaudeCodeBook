# Summary of `tools/SkillTool/`

## Purpose of `SkillTool/`

The `SkillTool/` directory implements Claude Code's **skill execution system**, enabling users to invoke predefined prompts and commands via slash syntax (e.g., `/commit`, `/review-pr`). The system handles prompt formatting, execution in either inline or forked contexts, permission management, telemetry, and terminal UI rendering.

## Contents Overview

| File | Role |
|------|------|
| `constants.ts` | Single exported constant `SKILL_TOOL_NAME = 'Skill'` — canonical identifier used throughout the codebase |
| `prompt.ts` | **Prompt generation engine** — formats skill listings, manages character budgets (1% of context window), truncates descriptions with bundled-skill priority, provides memoized `getPrompt()` output |
| `SkillTool.ts` | **Core tool implementation** — `validateInput`, `call`, execution logic (inline vs forked), permission checking, remote skill loading, telemetry emission |
| `UI.tsx` | **Terminal UI renderer** — Ink-based React components for results, progress, errors, and rejections |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                      constants.ts                        │
│           SKILL_TOOL_NAME = 'Skill'                      │
└─────────────────────┬───────────────────────────────────┘
                      │ Defines the tool name identifier
                      ▼
┌─────────────────────────────────────────────────────────┐
│                        prompt.ts                         │
│  • Receives command definitions from commands.js        │
│  • Formats skill listings within character budget        │
│  • Exports getPrompt() → used by SkillTool.call()        │
│  • Exports getCharBudget() → calculates context limits   │
│  • Exports clearPromptCache() → resets memoization       │
└─────────────────────┬───────────────────────────────────┘
                      │ Generates instruction string
                      ▼
┌─────────────────────────────────────────────────────────┐
│                     SkillTool.ts                         │
│  • Validates skill name (strips '/', handles remote)     │
│  • Checks permissions (SAFE_SKILL_PROPERTIES allowlist) │
│  • Executes inline → injects via contextModifier         │
│  • Executes forked → calls runAgent() in sub-agent      │
│  • Emits telemetry (skill_tool_invocation events)       │
│  • Returns Output with status: 'inline' | 'forked'       │
└─────────────────────┬───────────────────────────────────┘
                      │ Produces Output data
                      ▼
┌─────────────────────────────────────────────────────────┐
│                        UI.tsx                            │
│  • renderToolResultMessage → shows allowed tools, model  │
│  • renderToolUseProgressMessage → streams progress       │
│  • renderToolUseErrorMessage → displays errors           │
│  • renderToolUseRejectedMessage → shows rejections      │
└─────────────────────────────────────────────────────────┘
```

**Execution flow summary:**

1. User invokes `/skill-name` or `skill-name` in conversation
2. `SkillTool.validateInput()` normalizes the name and finds the command
3. Permission check against `SAFE_SKILL_PROPERTIES` allowlist
4. **Inline path**: `prompt.ts` formats instructions → injected via `contextModifier`
5. **Forked path**: `runAgent()` spins up isolated sub-agent → `runAgent.js` handles execution
6. Results flow through `UI.tsx` renderers for terminal display
7. Telemetry emitted to analytics (ant users only)

## Key Takeaways

- **Dual execution modes**: Inline skills preserve conversation context; forked skills run in isolated agents with separate token budgets — chosen when a skill has an `agent` definition
- **Bundled skill priority**: Skills with `type === 'prompt' && source === 'bundled'` are **never truncated**; they always display full descriptions even under tight context budgets
- **Permission safety model**: New `PromptCommand` properties default to requiring user permission; must be explicitly added to `SAFE_SKILL_PROPERTIES` to be auto-approved
- **Remote canonical skills** (experimental, ant-only): Load skill prompts from GCS/S3/HTTP via `_canonical_<slug>` naming convention, with `${CLAUDE_SKILL_DIR}` path substitution
- **Budget awareness**: Prompt generation accounts for context window size (via `contextWindowTokens`), defaulting to 8,000 characters if unknown; hard cap of 250 chars per description entry
- **Analytics gating**: Telemetry for non-ant users is sent unredacted; ant users get `_PROTO_*` field routing to preserve privacy
