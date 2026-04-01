# Summary of `tools/REPLTool/`

## Purpose of `REPLTool/`

Provides the core infrastructure for **REPL (Read-Eval-Print Loop) mode** in the Claude CLI. REPL mode controls tool availability—hiding direct tool access when enabled (forcing batch operations through the REPL interface) while still allowing those same tools to execute within the REPL's virtual machine context.

---

## Contents Overview

| File | Responsibility |
|------|----------------|
| `constants.ts` | Defines the `REPL` tool identifier, REPL-mode detection logic (environment-variable-driven), and the `REPL_ONLY_TOOLS` set of tools to hide when REPL mode is active |
| `primitiveTools.ts` | Exports a lazy-initialized array of 8 tool classes (FileRead, FileWrite, FileEdit, Glob, Grep, Bash, NotebookEdit, Agent) that are hidden from direct model calls but remain available inside the REPL VM |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                  REPLTool Directory                      │
├──────────────────────────┬──────────────────────────────┤
│     constants.ts         │      primitiveTools.ts        │
│                          │                               │
│  • REPL_TOOL_NAME        │  • getReplPrimitiveTools()    │
│  • isReplModeEnabled()   │  • _primitiveTools (cache)    │
│  • REPL_ONLY_TOOLS Set   │  • 8 Tool class references    │
│                          │                               │
│  ┌────────────────────┐  │                               │
│  │ Imports tool names │──┼──► Uses REPL_ONLY_TOOLS       │
│  └────────────────────┘  │   to know which tools to      │
│                          │   hide from direct use        │
└──────────────────────────┴──────────────────────────────┘
```

**Key relationship**: `constants.ts` imports tool **names** (strings) to build the `REPL_ONLY_TOOLS` set, while `primitiveTools.ts` imports the actual tool **classes** to provide the implementations accessible within the REPL VM.

**Consumer flow**:

1. **`isReplModeEnabled()`** (constants) is called at execution time to decide whether to filter tools
2. `REPL_ONLY_TOOLS` identifies which tool names should be hidden
3. `getReplPrimitiveTools()` (primitiveTools) provides the full tool implementations for the REPL VM to use internally

---

## Key Takeaways

| Takeaway | Details |
|----------|---------|
| **Lazy initialization** | `primitiveTools.ts` uses lazy initialization to break circular import chains (avoiding "Cannot access before initialization" errors) |
| **Mode-aware tool filtering** | The system supports three REPL states: forced on (`CLAUDE_REPL_MODE=1`), forced off (`CLAUDE_CODE_REPL=0`), and auto-detected based on `USER_TYPE` and `CLAUDE_CODE_ENTRYPOINT` |
| **Tool hiding ≠ tool removal** | Even when REPL mode is active and tools are hidden from the model's direct calls, they remain executable inside the REPL VM context |
| **SDK vs. CLI default** | REPL mode defaults to **off** for SDK entrypoints (exposing direct tool calls) and **on** for the ant-native CLI binary |
| **Circular dependency solution** | The lazy getter pattern in `primitiveTools.ts` resolves the circular dependency chain involving `collapseReadSearch.ts`, tool classes, and the tool registry |
