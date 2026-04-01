# Summary of `commands/hooks/`

## Purpose of `commands/hooks/`

This directory implements a **Tengu CLI command** that provides a UI for configuring hooks for tool events. It consists of a command definition layer and an implementation layer that work together to present a hooks configuration menu to users.

## Contents Overview

| File | Role | Key Responsibility |
|------|------|-------------------|
| `index.ts` | Command registry | Defines the command metadata (`name`, `type`, `description`) and lazy-loads the implementation |
| `hooks.tsx` | Command implementation | Contains the actual logic and UI rendering using `HooksConfigMenu` component |

### `index.ts` - Command Definition
```typescript
const hooks = {
  type: 'local-jsx',        // Renders a local JSX component
  name: 'hooks',            // Command identifier
  description: 'View hook configurations for tool events',
  immediate: true,          // Executes immediately when invoked
  load: () => import('./hooks.js'),  // Lazy-loads implementation
} satisfies Command
```
- Acts as the **entry point** for the command
- Uses dynamic import to defer loading until needed
- Validated against `Command` interface at compile-time

### `hooks.tsx` - Implementation
```typescript
export const call: LocalJSXCommandCall = async (onDone, context) => {
  // 1. Log analytics event
  logEvent('tengu_hooks_command', {});
  
  // 2. Extract permission context from app state
  const permissionContext = context.getAppState().toolPermissionContext;
  
  // 3. Get available tools and extract their names
  const toolNames = getTools(permissionContext).map(tool => tool.name);
  
  // 4. Render configuration menu
  return <HooksConfigMenu toolNames={toolNames} onExit={onDone} />;
};
```
- Contains the **business logic** and UI rendering
- Logs analytics on every invocation
- Queries available tools based on user permissions
- Passes data to `HooksConfigMenu` component

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────┐
│                    User invokes                          │
│                    "hooks" command                       │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              index.ts (Command Registry)                 │
│  • Type: 'local-jsx'                                    │
│  • Name: 'hooks'                                        │
│  • immediate: true                                     │
│  • load: () => import('./hooks.js')                     │
└─────────────────────┬───────────────────────────────────┘
                      │ Dynamic import (lazy load)
                      ▼
┌─────────────────────────────────────────────────────────┐
│              hooks.tsx (Implementation)                  │
│  • logEvent() ──────────────────────► Analytics         │
│  • context.getAppState()            │                  │
│  • getTools(permissionContext)      │                  │
│  • <HooksConfigMenu /> ─────────────┼───► UI Rendered   │
│    ├─ toolNames: string[]           │                  │
│    └─ onExit: onDone callback ──────┘                  │
└─────────────────────────────────────────────────────────┘
```

**Flow:**
1. User invokes `hooks` command → CLI loads `index.ts`
2. `index.ts` triggers `load()` → lazy imports `hooks.tsx`
3. `hooks.tsx` executes `call()` function:
   - Logs analytics event
   - Retrieves available tools via `getTools()`
   - Renders `HooksConfigMenu` with tool names
4. User interacts with menu → `onDone` callback triggers → command completes

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Pattern** | Lazy-loaded local JSX command with analytics |
| **Data flow** | App state → Tool permissions → Available tools → UI component |
| **Security** | Tool access controlled by `toolPermissionContext` |
| **Analytics** | Every invocation logs `'tengu_hooks_command'` event |
| **UI component** | Delegates rendering to shared `HooksConfigMenu` component |
| **Type safety** | Uses `satisfies Command` and typed `LocalJSXCommandCall` signature |

### Design Principles Observed:
- **Separation of concerns**: Registry (`index.ts`) vs. implementation (`hooks.tsx`)
- **Lazy loading**: Implementation loaded only when command invoked
- **Immediate execution**: `immediate: true` means no user prompting before render
- **Permission-aware**: Filters tools based on user's permission context
