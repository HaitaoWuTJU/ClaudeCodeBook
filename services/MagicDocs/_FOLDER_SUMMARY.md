# Summary of `services/MagicDocs/`

## Purpose of `MagicDocs/`

The `MagicDocs/` directory implements **Magic Docs**, a self-maintaining documentation system for Claude Code. When a markdown file is marked with a `# MAGIC DOC: [title]` header and read by an "ant" user, the system automatically updates the document with learnings from the conversation—keeping documentation fresh without manual effort.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Core engine: detects magic doc headers, registers files, orchestrates background updates via a forked subagent with restricted tool access |
| `prompts.ts` | Prompt template infrastructure: loads or defaults to update prompts, performs variable substitution |

## How Files Relate to Each Other

```
MagicDocs/
┌─────────────────┐
│   index.ts      │  ← Orchestrator
│                 │
│  • Detects header via MAGIC_DOC_HEADER_PATTERN
│  • Tracks files in trackedMagicDocs Map
│  • Hooks into post-sampling (idle detection)
│  • Calls updateMagicDoc() per tracked file
│                 │
└────────┬────────┘
         │ calls
         ↓
┌──────────────────────────────────────────────────────────────┐
│                        prompts.ts                            │
│  • Provides update instructions (custom ~/.claude/... or default)
│  • Substitutes {{docContents}}, {{docPath}}, {{docTitle}}
│  • Returns fully composed prompt string                      │
└──────────────────────────────────────────────────────────────┘
```

The **flow** is:
1. `registerFileReadListener` (in `index.ts`) fires when a file is read
2. `detectMagicDocHeader()` extracts title and optional instructions
3. `registerMagicDoc()` adds the file to `trackedMagicDocs`
4. After each assistant response, `updateMagicDocs` runs **if** conversation is idle
5. For each tracked doc, `updateMagicDoc()` reads current content, builds a prompt via `buildMagicDocsUpdatePrompt()` (from `prompts.ts`), and runs a forked subagent restricted to editing only that file

## Key Takeaways

- **Zero-config activation**: Magic Docs are detected automatically—just add `# MAGIC DOC: [title]` to any markdown file
- **Restricted execution context**: The subagent can only use `FileEditTool` on the specific magic doc path, preventing accidental side effects
- **Idle-aware updates**: The hook only fires when the conversation has no tool calls in the last turn, avoiding interference with ongoing tasks
- **Customizable prompts**: Users can override the update instructions by placing a custom prompt at `~/.claude/magic-docs/prompt.md`
- **Graceful degradation**: Missing/inaccessible files are silently removed from tracking; non-ant users are unaffected
- **State isolation**: Uses `cloneFileStateCache()` to prevent stale file-state stubs from blocking fresh reads during updates
