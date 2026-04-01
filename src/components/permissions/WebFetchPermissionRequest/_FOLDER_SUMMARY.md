# Summary of `components/permissions/WebFetchPermissionRequest/`

## Purpose of `WebFetchPermissionRequest/`

This directory contains a single component responsible for rendering a user-facing permission dialog when Claude attempts to fetch web content. Its purpose is to:

1. **Inform** the user about a pending web fetch request (displaying the target URL and domain)
2. **Offer choices** between a one-time allow, a permanent domain-level allow rule, or rejection
3. **Persist preferences** when users opt for "don't ask again for this domain"
4. **Log** permission decisions for auditing or analytics

## Contents Overview

| File | Description |
|------|-------------|
| `WebFetchPermissionRequest.jsx` | The sole file in this directory. Exports a React component and a pure utility function. |

### `WebFetchPermissionRequest.jsx` at a glance

```
inputToPermissionRuleContent(input)
  └─ Extracts hostname from URL → returns "domain:{hostname}"

WebFetchPermissionRequest(props)
  ├─ useMemo → extracts URL, hostname, domain rule, input validation
  ├─ PermissionDialog
  │   ├─ Title: "Claude wants to fetch this URL"
  │   ├─ Rule: "domain:{hostname}"
  │   └─ Options (via <Select>)
  │       ├─ "Yes" → onAllow([])
  │       ├─ "Yes, don't ask again for {domain}" → onAllow([rule], behavior=allow)
  │       └─ "No" → onReject()
  └─ Logs permission event via logUnaryPermissionEvent()
```

## How Files Relate to Each Other

```
WebFetchPermissionRequest.jsx
  │
  ├── imports WebFetchTool.inputSchema (for URL validation)
  ├── imports PermissionDialog (layout shell)
  ├── imports Select (interactive choice dropdown)
  ├── imports shouldShowAlwaysAllowOptions (opt-in flag)
  ├── imports logUnaryPermissionEvent (analytics)
  ├── imports usePermissionRequestLogging (hooks)
  ├── imports useTheme (styling via Ink)
  └── reads/writes toolUseConfirm (parent-supplied state)
```

There are **no subdirectories** within this folder. All logic is co-located in a single JSX file. The component is a leaf node in the component tree — it is consumed by a parent permission orchestration layer and receives all necessary data via props (primarily `toolUseConfirm`).

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Single responsibility** | This component only handles web fetch permission UX; it does not fetch data itself. |
| **Domain-level persistence** | When users choose the "don't ask again" option, a rule is written to `localSettings` with `destination: "localSettings"`, meaning the preference survives across sessions. |
| **Host-based rules** | Rules are hostname-scoped (e.g., `domain:example.com`), not full-URL-scoped. This grants broad permission for any path on the same domain. |
| **Conditional UX** | The permanent-allow option is gated behind `shouldShowAlwaysAllowOptions()`, allowing admins to disable that opt-in for security-sensitive environments. |
| **React compiler ready** | The component uses the React compiler runtime (`_c`, symbol-sentinel memoization), indicating it is optimized for future automatic memoization. |
| **Cancel = Reject** | Pressing `Esc` during selection is treated the same as explicitly choosing "No" — the fetch is blocked. |
