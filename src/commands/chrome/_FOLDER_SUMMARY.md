# Summary of `commands/chrome/`

## Purpose of `chrome/`

The `chrome/` directory implements the **Claude in Chrome** feature for Claude Code — a CLI extension management system that allows users to install, configure, and manage the official Claude Chrome extension directly from the terminal.

## Contents Overview

| File | Role |
|------|------|
| `index.ts` | Command registration entry point — defines the `chrome` command metadata, availability, enablement logic, and lazy-loads the implementation |
| `chrome.tsx` | Main UI component — renders a full-featured dialog for extension status, installation, permission management, and settings |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                     chrome/index.ts                         │
│  - Command definition (name, description, availability)     │
│  - Lazy-loads ./chrome.js via dynamic import()              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     chrome/chrome.tsx                       │
│  - Main React component (ClaudeInChromeMenu)                │
│  - Renders Dialog with status indicators and menu options   │
│  - Reads: extension status, config, subscription, MCP state  │
│  - Writes: updates global config for default enable pref    │
└─────────────────────────────────────────────────────────────┘
```

**Flow**: When a user runs the `chrome` command, `index.ts` dynamically imports `chrome.tsx`, which renders the interactive menu and handles all user interactions (install extension, manage permissions, reconnect, toggle default).

## Key Takeaways

- **Feature gated** by `claude-ai` subscription and platform (disabled in WSL, non-interactive sessions)
- **React Compiler** is used with 41 cache slots for performance optimization
- **Lazy loading** keeps the main bundle lean — the chrome implementation is only loaded on demand
- **Dynamic URL resolution** — opens Claude URLs in Chrome on Homespace, in default browser elsewhere
- **MCP integration** — the menu displays the connection state of the internal `claude-in-chrome` MCP server
