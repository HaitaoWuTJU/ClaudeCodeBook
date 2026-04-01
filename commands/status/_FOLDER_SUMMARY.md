# Summary of `commands/status/`

## Purpose of `status/`

Provides a command module that displays Claude Code's status information, including version, model, account details, API connectivity, and tool statuses. The feature is accessed via the `status` command and renders within a Settings panel.

---

## Contents Overview

| File | Role | Purpose |
|------|------|---------|
| `status.ts` | Command registration | Configuration object that declares the command metadata and lazy-loads the implementation |
| `status.jsx` | Command implementation | React component entry point that renders the Settings panel with the Status tab |

---

## How Files Relate to Each Other

```
status.ts                         status.jsx
┌─────────────────────────┐       ┌──────────────────────────────┐
│  type: 'local-jsx'      │       │  call(onDone, context) {     │
│  name: 'status'         │       │    return <Settings          │
│  load: () => import(...)│──────►│      defaultTab="Status"      │
└─────────────────────────┘       │    />                        │
                                   └──────────────────────────────┘
```

1. **`status.ts`** acts as the **command descriptor** — it declares metadata (name, description, type) and uses a dynamic import in `load()` to defer loading the actual implementation.

2. **`status.jsx`** provides the **command executor** — the `call` function is the standard command interface that receives the `onDone` callback and `context`, then returns a React node (the Settings component).

3. The **lazy loading pattern** in `status.ts` keeps the initial bundle lean; `status.jsx` is only loaded when the command is actually invoked.

---

## Key Takeaways

- **Lazy loading** via `load()` dynamic import optimizes startup performance
- **`immediate: true`** suggests this command bypasses normal async command queuing
- **Tab routing** via `defaultTab="Status"` opens the Settings panel directly to the relevant view
- **Command interface pattern** — the codebase uses a consistent `call(onDone, context)` signature for local JSX commands
