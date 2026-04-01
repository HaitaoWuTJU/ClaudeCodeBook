# Summary of `components/permissions/SkillPermissionRequest/`

# `src/components/permissions/SkillPermissionRequest/`

## Purpose of `SkillPermissionRequest/`

This directory contains the React component responsible for rendering permission dialogs when Claude Desktop attempts to execute a Skill tool. It provides users with granular control over skill permissions, including options to allow execution once, or to configure persistent "always allow" rules with different matching granularity.

## Contents Overview

| File | Purpose |
|------|---------|
| `index.js` | Main entry point exporting the `SkillPermissionRequest` component |
| `SkillPermissionRequest.js` | Core component implementing the permission dialog with multiple response options |
| `SkillPermissionRequest.module.css` | (Referenced) Styling for the component |

## How Files Relate to Each Other

```
SkillPermissionRequest.js
├── Imports: utils/log.js, bootstrap/state.js, permissions/permissionsLoader.js
├── Imports: SkillTool for inputSchema validation
├── Imports: PermissionDialog, PermissionPrompt, PermissionRuleExplanation
├── Exports: SkillPermissionRequest component
└── index.js
    └── Re-exports SkillPermissionRequest from SkillPermissionRequest.js
```

**Data Flow Diagram:**

```
toolUseConfirm.input
         │
         ▼
   parseInput()
   (validates via SkillTool.inputSchema)
         │
         ▼
  alwaysAllowOptions ──────────────────┐
  (exact, prefix, base)                │
         │                             │
         ▼                             │
   PermissionPrompt ◄─────────────────┤
         │                             │
    handleSelect() ────────────────────┤
    handleCancel()                     │
         │                             │
         ▼                             ▼
  toolUseConfirm.onAllow()    logUnaryEvent()
  toolUseConfirm.onReject()   (analytics)
         │
         ▼
  onDone() / onReject()
  (callbacks)
```

## Key Takeaways

1. **Granular Permission Control**: The component supports three permission levels—"always allow exact match," "always allow by command prefix," and "allow once"—providing users with fine-grained control over skill execution permissions.

2. **React Compiler Optimization**: Uses React 19+ compiler runtime with a 51-slot cache array for memoizing derived values like `alwaysAllowOptions` and `currentInput`, optimizing re-render performance.

3. **Input Validation**: Leverages `SkillTool.inputSchema.safeParse()` to validate tool inputs before processing, ensuring only properly formatted skill requests are handled.

4. **Analytics Integration**: Every user action (accept/reject) triggers `logUnaryEvent` with platform, message ID, completion type, and language metadata for usage tracking.

5. **MCP Tool Support**: Distinguishes between native skills and MCP (Model Context Protocol) tools via `toolUseConfirm.tool.isMcp` flag for analytics context.

6. **Local Settings Integration**: "Always allow" options persist rules to `localSettings` with a structured format: `{ toolName: "skill", ruleContent, behavior: "allow", destination: "localSettings" }`.
