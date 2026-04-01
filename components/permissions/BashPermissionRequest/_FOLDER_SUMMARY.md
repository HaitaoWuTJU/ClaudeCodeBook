# Summary of `components/permissions/BashPermissionRequest/`

## BashPermissionRequest Directory Summary

### Purpose

The `BashPermissionRequest/` directory provides the permission workflow for bash command execution in a CLI agent tool. When the agent attempts to run shell commands, this directory handles the interactive confirmation dialog, destructive command warnings, sandboxing decisions, and the generation of persistent "always allow" permission rules. It bridges the gap between raw bash execution and the agent's permission/safety system.

---

### Contents Overview

| File | Purpose |
|------|---------|
| `BashPermissionRequest.tsx` | Main component — parses bash tool input, delegates sed edits, orchestrates hooks, and renders the `PermissionDialog` with options |
| `bashToolUseOptions.tsx` | Builds the selectable Yes/No permission options, including "always allow" rule generation with descriptions |

---

### How Files Relate to Each Other

```
BashPermissionRequest.tsx
│
├── parses BashTool.inputSchema
│       │
│       ├── is sed edit? ──► delegates to SedEditPermissionRequest
│       │
│       └── is bash ──► BashPermissionRequestInner
│                          │
│                          ├── useShellPermissionFeedback()
│                          ├── usePermissionExplainerUI()
│                          ├── generateGenericDescription()
│                          │       ├── getSimpleCommandPrefix() [sync]
│                          │       └── getCompoundCommandPrefixesStatic() [async/tree-sitter]
│                          │
│                          ├── isDestructiveWarning? ──► show warning banner
│                          ├── isSandboxed? ──► show sandbox badge
│                          │
│                          └── bashToolUseOptions.tsx ←── generates permission options
│                                  │
│                                  ├── always-allow rules (if shouldShowAlwaysAllowOptions)
│                                  ├── editable prefix input (if editablePrefix)
│                                  ├── Haiku suggestions (if suggestions exist)
│                                  └── classifier option (if isClassifierPermissionsEnabled)
```

`BashPermissionRequest.tsx` owns the data-fetching hooks (classifier description, destructive/sandbox checks) and passes the results into `bashToolUseOptions()`, which assembles the static option list. The options are then rendered inside `PermissionDialog` as a `Select` component.

---

### Key Takeaways

1. **Multi-stage description generation** — The bash command description starts with a synchronous `getSimpleCommandPrefix()` call (first word of the command, with sandbox indicator), then is asynchronously refined by the classifier with tree-sitter compound-command analysis.

2. **Shimmer animation isolation** — `ClassifierCheckingSubtitle` was extracted into its own component to prevent the 20fps clock tick (used by `useShimmerAnimation`) from re-rendering the entire `PermissionDialog` + `Select` tree during the 1-3 second classifier check window.

3. **Compound command prefix is editable** — The backend provides per-subcommand rule suggestions (e.g., `cd src:* && cd tests:*` → two separate rules). When exactly one rule is suggested, `editablePrefix` is seeded from it so the user can refine it. When zero or multiple rules exist, no editable prefix is shown, preventing dead rules (GH#11380).

4. **Sed edit delegation** — Sed-based edits are immediately delegated to `SedEditPermissionRequest` rather than being processed through the bash permission flow, likely due to different UI/rule-generation semantics.

5. **Classifier permissions as a guard** — The `yes-classifier-reviewed` option (for auto-approval) only appears when `isClassifierPermissionsEnabled` is true and the decision reason is not a server-side classifier block. Classifier descriptions are deduplicated against existing allow rules.

6. **Destructive/sandbox warnings are feature-flagged** — Both the destructive command warning banner and sandbox indicator are gated behind `tengu_destructive_command_warning` and `SandboxManager.isSandboxingEnabled()` respectively.
