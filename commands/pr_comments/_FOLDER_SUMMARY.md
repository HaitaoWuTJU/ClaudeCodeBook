# Summary of `commands/pr_comments/`

## Purpose of `pr_comments/`

This directory defines a CLI command (`pr-comments`) that fetches and displays all comments from a GitHub pull request. It supports two types of comments: PR-level discussion comments and code review comments with diff context.

## Contents Overview

**`index.ts`** - Single file command definition containing:

| Component | Purpose |
|-----------|---------|
| `name: 'pr-comments'` | Command identifier |
| `description` | Human-readable CLI help text |
| `progressMessage` | Status displayed during execution |
| `getPromptWhileMarketplaceIsPrivate()` | AI prompt instructing how to fetch and format PR comments using `gh` CLI |

## How Files Relate to Each Other

```
commands/pr_comments/
└── index.ts ── imports ──► ../createMovedToPluginCommand.js
                                   │
                                   └─► Provides factory for command creation
                                        and plugin migration logic
```

The directory contains only `index.ts`, which acts as a thin wrapper that:
1. Imports the `createMovedToPluginCommand` factory from the parent `commands/` directory
2. Passes configuration and the prompt function to the factory
3. Exports the resulting command object

## Key Takeaways

- **Plugin migration path**: This command uses a factory pattern to smoothly transition from inline AI-based execution to a dedicated plugin (`pr-comments`)
- **Dual data sources**: Fetches from both GitHub Issues API (`/issues/{number}/comments`) for PR discussions and Pull Requests API (`/pulls/{number}/comments`) for code review comments
- **Context preservation**: The AI prompt instructs fetching actual file content via `gh api` so comments can be displayed with their referenced code context
- **User customization**: Optional command arguments are appended to the AI prompt for flexible behavior
- **No local dependencies**: The file itself has no internal dependencies beyond the parent factory; all data fetching happens via the `gh` CLI invoked by the AI agent
