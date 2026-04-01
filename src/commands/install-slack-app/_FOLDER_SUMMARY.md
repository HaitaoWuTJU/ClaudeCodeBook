# Summary of `commands/install-slack-app/`

## Purpose of `install-slack-app/`

Provides a local Claude command that redirects users to the Slack marketplace to install the Claude Slack app, while tracking installation events.

## Contents Overview

| File | Role |
|------|------|
| `commands/install-slack-app/install-slack-app.ts` | Command descriptor (registry entry with lazy-loading) |
| `commands/install-slack-app.ts` | Command implementation (business logic) |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│  commands/install-slack-app/install-slack-app.ts            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Command Descriptor (lazy-loads implementation)        │  │
│  │ • type: 'local'                                       │  │
│  │ • name: 'install-slack-app'                          │  │
│  │ • load: () => import('./install-slack-app.js')       │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            │ dynamic import()               │
│                            ▼                                │
│  commands/install-slack-app.ts                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Command Implementation (actual business logic)        │  │
│  │ • Opens Slack marketplace URL                         │  │
│  │ • Logs analytics event                                │  │
│  │ • Increments install counter                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

The descriptor file acts as a **lazy-loading proxy**: it defines command metadata for the CLI to register, then defers all execution to the implementation file until the command is actually invoked.

## Key Takeaways

1. **Lazy Loading Pattern**: The descriptor uses `import()` to dynamically load the implementation, reducing the initial bundle size
2. **Availability Restriction**: The command only works in the `claude-ai` environment
3. **Interactive-Only**: The command requires user interaction (`supportsNonInteractive: false`)
4. **Analytics Tracking**: Every invocation logs a `tacku_install_slack_app_clicked` event
5. **State Persistence**: Tracks cumulative install clicks in global config via `slackAppInstallCount`
6. **Graceful Degradation**: Returns a manual URL if the browser fails to open
