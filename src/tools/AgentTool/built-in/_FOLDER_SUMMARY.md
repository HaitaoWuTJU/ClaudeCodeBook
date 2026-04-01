# Summary of `tools/AgentTool/built-in/`

## Purpose of `built-in/`

This directory houses pre-configured, special-purpose agents for Claude Code CLI that extend the main agent's capabilities. Each agent has a narrow focus, specific tool restrictions, and tailored system prompts enabling autonomous execution of focused tasks—from file exploration to verification.

## Contents Overview

| File | Agent Type | Primary Role | Model | Key Restriction |
|------|------------|--------------|-------|-----------------|
| `claudeCodeGuideAgent.ts` | `'claude-code-guide'` | Documentation lookup for CLI/SDK/API | `'haiku'` | Read-only docs |
| `exploreAgent.ts` | `'explore'` | Fast file search & codebase navigation | `'haiku'` or `'inherit'` | Read-only |
| `generalPurposeAgent.ts` | `'general-purpose'` | Multi-step code research & analysis | Default subagent model | No file creation |
| `planAgent.ts` | `'plan'` | Architecture & implementation planning | `'inherit'` | Read-only |
| `statuslineSetup.ts` | `'statusline-setup'` | Shell PS1 → Claude Code status conversion | `'sonnet'` | Edit settings only |
| `verificationAgent.ts` | `'verification'` | Pre-merge implementation correctness checks | Inherited | No project modifications |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                     Built-in Agent Registry                      │
│                 (loaded via loadAgentsDir.js)                    │
└─────────────────────────────────────────────────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
┌─────────────────┐ ┌──────────┐ ┌─────────────┐ ┌────────────────┐
│  Plan Agent     │ │ Explore  │ │ Verification│ │ ClaudeCodeGuide│
│  (architecture) │ │ Agent    │ │ Agent       │ │ Agent          │
└────────┬────────┘ └────┬─────┘ └──────┬──────┘ └───────┬────────┘
         │               │               │                │
         ▼               ▼               ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Shared Dependencies                           │
│  • Tool name constants (Bash, Glob, Grep, Read, Write, Edit)    │
│  • hasEmbeddedSearchTools() — switches Glob/Grep ↔ find/grep   │
│  • BuiltInAgentDefinition type (from loadAgentsDir.js)          │
└─────────────────────────────────────────────────────────────────┘
```

**Key relationships:**

- **`exploreAgent.ts` → `planAgent.ts`**: The Plan Agent reuses `EXPLORE_AGENT.tools` as its foundation
- **`hasEmbeddedSearchTools()`**: Shared utility in `src/utils/embeddedTools.js` used by `exploreAgent`, `planAgent`, and `claudeCodeGuideAgent` to conditionally adapt tool configurations for Ant-native builds
- **`generalPurposeAgent.ts`**: Standalone; no interdependencies; designed for future enhancement via `enhanceSystemPromptWithEnvDetails`
- **`statuslineSetup.ts`**: Siloed; only tool dependency is `FILE_READ_TOOL_NAME` / `FILE_EDIT_TOOL_NAME`; not used by other agents
- **`verificationAgent.ts`**: Independent; explicitly forbids spawning any sub-agent (self-reference prevention via disallowed `AGENT_TOOL_NAME`)

## Key Takeaways

1. **Tiered tool access**: Agents use `'*'` (general-purpose), explicit arrays of specific tools (explore, plan, guide), or tool restrictions (verification — ephemeral temp writes only). All read-only agents share the same `omitClaudeMd: true` optimization.

2. **Embedded search duality**: Multiple agents conditionally route between native `Glob`/`Grep` tools and `find`/`grep` aliases depending on whether the host environment has embedded search tools. This is the most common shared behavioral branch in the directory.

3. **Verification as gatekeeper**: The verification agent enforces a strict no-project-modification contract, requiring ephemeral scripts to go to `/tmp` only. Its `VERDICT: PASS/FAIL/PARTIAL` output format is machine-parseable.

4. **Configuration vs. capability**: `statuslineSetup` is the only agent whose purpose is *configuration* (converting shell PS1 to `settings.json`) rather than code tasks. It receives structured JSON via stdin (session metadata, rate limits, context usage).

5. **Model selection hierarchy**: 
   - Explicit `'haiku'` (explore, guide) for lightweight/speed
   - `'sonnet'` (statusline) for complex config parsing
   - `'inherit'` (plan, explore-ant) for alignment with parent
   - `generalPurposeAgent` intentionally omits model for default resolution

6. **Documentation-first architecture**: The `claudeCodeGuideAgent` fetches remote documentation maps at runtime and is the only agent that depends on `WEB_FETCH_TOOL_NAME` for non-local knowledge.
