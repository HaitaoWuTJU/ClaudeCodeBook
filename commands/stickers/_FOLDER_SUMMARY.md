# Summary of `commands/stickers/`

## Purpose of `stickers/`

Implements a CLI command that opens the Sticker Mule promotional page for Claude Code stickers in the user's default browser, enabling users to order stickers with a single command.

## Contents Overview

| File | Role | Purpose |
|------|------|---------|
| `stickers.ts` | Command descriptor | Registers command metadata (name, description, lazy-load path) |
| `index.ts` | Command handler | Contains the actual execution logic (opens URL in browser) |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────┐
│                   stickers.ts                       │
│  • Exported as default export                       │
│  • Provides metadata + load function                │
│  • load() returns import('index.ts')               │
└───────────────────────┬─────────────────────────────┘
                        │ on-demand import
                        ▼
┌─────────────────────────────────────────────────────┐
│                     index.ts                        │
│  • Exports call() async function                    │
│  • openBrowser() opens sticker URL                  │
│  • Returns LocalCommandResult with user message    │
└─────────────────────────────────────────────────────┘
```

The **separation of concerns** follows a common CLI pattern:

1. `stickers.ts` acts as the **entry point/registry** — lightweight, always loaded
2. `index.ts` contains the **heavy logic** — only loaded when the user runs `stickers`
3. This lazy-loading pattern keeps the CLI startup fast

## Key Takeaways

- **Interactive-only**: `supportsNonInteractive: false` means this command cannot run in CI environments
- **Type safety**: Both files use TypeScript interfaces (`Command`, `LocalCommandResult`) for compile-time validation
- **Graceful degradation**: If the browser fails to open, the command returns the URL so users can still visit manually
- **Partner integration**: Links to Sticker Mule, suggesting a promotional partnership for physical merchandise
