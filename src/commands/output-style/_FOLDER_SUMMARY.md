# Summary of `commands/output-style/`

## Purpose of `output-style/`

This directory implements a deprecated command handler for `/output-style`. Its sole purpose is to notify users that the command has been deprecated and redirect them to use `/config` instead.

## Contents Overview

| File | Purpose |
|------|---------|
| `output-style.ts` | Command configuration file that registers the command with metadata (hidden, deprecated, lazy-loaded) |
| `output-style.tsx` | Command implementation that displays a deprecation notice via system message |

## How Files Relate to Each Other

```
output-style.ts (config)
       │
       │ lazy-loads via import()
       ▼
output-style.tsx (implementation)
       │
       │ exports call() function
       ▼
   invoked by command system
```

The relationship follows the lazy-loading pattern established by the parent command system:

1. **`output-style.ts`** acts as the entry point and metadata descriptor. It tells the command system:
   - The command exists and is hidden from help (`isHidden: true`)
   - It's deprecated (`description` warns users)
   - The actual implementation lives in `output-style.tsx` and should be loaded on demand

2. **`output-style.tsx`** contains the actual runtime behavior. When loaded and invoked, it calls the `onDone` callback with a deprecation message and `display: 'system'` to show a system notification to the user.

## Key Takeaways

- **Deprecated feature** — This command existed to change output style but is no longer the preferred approach
- **Lazy loading** — Implementation is separated and only loaded when the command is actually invoked, keeping the main bundle lean
- **Hidden from users** — The command is not listed in help, but can still be invoked by users who know it exists
- **Migration path** — Users are directed to use `/config` for changing output style settings
- **System-level messaging** — The deprecation notice uses `display: 'system'`, indicating it's a framework/system message rather than a user message
