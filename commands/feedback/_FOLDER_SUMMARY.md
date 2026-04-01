# Summary of `commands/feedback/`

## Purpose of `feedback/`

Provides a complete command module for the "feedback" (and "bug") command in Claude Code CLI. Users can trigger this command to submit feedback or bug reports, which gets rendered as a React component within the CLI interface.

## Contents Overview

| File | Role |
|------|------|
| `feedback.tsx` | **Runtime implementation** — renders the `<Feedback />` React component, handles the async `call` entry point with abort signal support |
| `index.ts` | **Configuration & gatekeeper** — defines command metadata (name, aliases, type, description), determines whether the command is available based on environment/user/policy constraints, and lazily loads the runtime |

## How Files Relate to Each Other

```
index.ts (command config)
    │
    ├── isEnabled() checks:
    │   ├── Not on Bedrock / Vertex / Foundry
    │   ├── Not essential traffic only
    │   ├── User type ≠ 'ant'
    │   └── allow_product_feedback policy ✓
    │
    ▼
load() → dynamically imports → feedback.tsx
    │
    ▼
call(onDone, context, args?)
    │
    ▼
renderFeedbackComponent(onDone, signal, messages, initialDescription, backgroundTasks)
    │
    ▼
<Feedback /> (React component with JSX)
```

**Lifecycle:**

1. **At startup** — `index.ts` registers the command. No code from `feedback.tsx` is loaded yet.
2. **At invocation** — When the user types `feedback` or `bug`, `load()` is called, which `import()`s `feedback.tsx`.
3. **At execution** — `call()` receives the context (abort signal, messages), optionally parses an argument as `initialDescription`, and renders the React component via `renderFeedbackComponent()`.

## Key Takeaways

- **Zero-cost feature flag** — `index.ts` uses early-return `isEnabled()` checks before any heavy code loads. The feedback module is never parsed unless the command is permitted.
- **Lazy loading** — The `load()` function uses a dynamic `import()` so tree-shaking and lazy code splitting can exclude `feedback.tsx` from production bundles if the feature is disabled.
- **Cancellation support** — `feedback.tsx` passes `AbortSignal` through to the component, enabling CLI-level cancellation (e.g., user presses Ctrl+C) to propagate correctly.
- **Background task awareness** — The component receives a `backgroundTasks` map, allowing it to reference or display in-progress tasks alongside the feedback form.
- **Accessibility via alias** — `bug` is an alias so users can quickly report issues without needing to remember the exact command name.
