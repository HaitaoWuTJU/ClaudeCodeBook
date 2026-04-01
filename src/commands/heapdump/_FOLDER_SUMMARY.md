# Summary of `commands/heapdump/`

## Purpose of `heapdump/`

The `heapdump/` directory implements a hidden CLI command that captures a JavaScript heap dump and writes the resulting files to the user's Desktop. It is intended for debugging and diagnostics purposes.

## Contents Overview

The directory contains two files working together in a lazy-loading pattern:

| File | Role |
|------|------|
| `index.ts` | **Command descriptor** — defines metadata (name, hidden flag, etc.) and lazy-loads the implementation |
| `heapdump.ts` | **Command implementation** — orchestrates the heap dump via the heap dump service and formats the response |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│  index.ts (Command Descriptor)                          │
│  ├── type: 'local'                                      │
│  ├── name: 'heapdump'                                   │
│  ├── isHidden: true                                     │
│  └── load() → dynamic import('./heapdump.js')           │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  heapdump.ts (Implementation)                           │
│  ├── Imports: heapDumpService.performHeapDump()         │
│  ├── Calls: performHeapDump()                           │
│  └── Returns: { type: 'text', value: <paths|error> }    │
└─────────────────────────────────────────────────────────┘
```

1. When a user invokes `heapdump`, the command runner uses `index.ts` to register the command
2. The actual execution is deferred until `load()` is called, importing `heapdump.ts` on demand
3. `heapdump.ts` invokes `performHeapDump()` from the shared `heapDumpService` utility
4. The result (file paths or error) is returned in a structured format for the command runner

## Key Takeaways

- **Hidden command**: The `heapdump` command is intentionally hidden from the standard help output
- **Lazy loading**: Implementation is loaded only when invoked, keeping startup times fast
- **Service separation**: Core heap dump logic is centralized in `heapDumpService.js` for reuse
- **Structured response**: Output uses a consistent `{ type, value }` format suitable for piping or programmatic consumption
- **Non-interactive support**: Works in both interactive and CI/CD environments
