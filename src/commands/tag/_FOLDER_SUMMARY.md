# Summary of `commands/tag/`

## Purpose of `tag/`

Implements the `/tag` CLI command, enabling users to add searchable tags to the current session. Running the command twice with the same tag prompts to remove it, creating a toggle behavior.

## Contents Overview

```
tag/
├── index.tsx    # Command configuration (metadata, lazy loading, enable guard)
└── tag.tsx      # Command implementation (UI, logic, analytics)
```

| File | Role |
|------|------|
| `index.tsx` | Command entry point — defines metadata (name, description, enable condition) and lazy-loads implementation |
| `tag.tsx` | Implementation — renders interactive UI, handles tag add/remove logic, emits analytics events |

## How Files Relate to Each Other

```
┌─────────────────────────────────────┐
│         User runs /tag              │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  index.tsx (tag object)             │
│  - Validates isEnabled()            │
│  - Triggers lazy import('./tag.js') │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  tag.tsx (call function)            │
│  - Shows help if no args           │
│  - Adds tag directly if different   │
│  - Prompts confirm if same tag      │
└─────────────────────────────────────┘
```

- `index.tsx` acts as a **facade** — it satisfies the `Command` interface and defers all heavy lifting to the lazy-loaded `tag.tsx`
- `tag.tsx` contains all **business logic**: session management, sanitization, confirmation dialogs, and analytics
- No circular dependencies; `index.tsx` imports only types, not the implementation

## Key Takeaways

1. **Lazy loading** — Implementation code is not bundled until the command is actually invoked, keeping initial load time minimal
2. **User-type gating** — The command is hidden for non-`ant` users (`isEnabled` check on `USER_TYPE`)
3. **Toggle UX** — The same tag twice prompts removal; different tags replace silently
4. **Analytics-first** — Every user interaction (add, prompt, confirm, cancel) emits a named event for tracking
5. **Terminal UI primitives** — Uses Ink (`Box`, `Text`) and custom components (`Dialog`, `Select`) for rendering in the CLI environment
