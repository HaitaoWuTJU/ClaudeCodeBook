# Summary of `commands/btw/`

## Purpose of `btw/`

Implements the `/btw` (by the way) command for a terminal-based CLI agent tool. This command allows users to ask follow-up questions while the main agent is processing, displaying the response in a non-blocking side panel without interrupting the primary workflow.

## Contents Overview

| File | Type | Description |
|------|------|-------------|
| `index.ts` | TypeScript Config | Command registration metadata (name, description, lazy load) |
| `btw.jsx` | React Component | Interactive UI with keyboard navigation and markdown rendering |

## How Files Relate to Each Other

```
┌─────────────────┐     lazy-loads      ┌──────────────────┐
│   index.ts      │ ─────────────────► │     btw.jsx      │
│ (Command Config)│                    │ (React Component)│
└─────────────────┘                    └──────────────────┘
```

- **`index.ts`** acts as the entry point that registers the command with the CLI framework
- **`btw.jsx`** contains the actual implementation: fetches the side question response, renders the loading spinner, displays markdown output, and handles keyboard events (arrow keys, escape, enter)
- When `/btw <question>` is invoked, the framework:
  1. Loads `btw.jsx` dynamically
  2. Calls the `call()` export with the question
  3. Renders the `BtwSideQuestion` component with the fetched response

## Key Takeaways

1. **Lazy Loading Pattern**: The actual component is only loaded when the command is invoked, keeping the initial bundle smaller

2. **Prompt Cache Optimization**: The implementation includes a sophisticated caching strategy (`buildCacheSafeParams`) that reuses byte-identical request parameters from the main thread to maximize prompt cache hit rates

3. **Keyboard-First UX**: Full keyboard navigation support (↑/↓ scroll, escape/enter/space dismiss) for terminal-native interaction

4. **Non-Blocking**: The `immediate: true` flag and separate rendering path ensures the side question doesn't block the main agent's execution

5. **React Compiler Compatible**: Uses `react/compiler-runtime` with cache sentinels, indicating compliance with React's compiler requirements
