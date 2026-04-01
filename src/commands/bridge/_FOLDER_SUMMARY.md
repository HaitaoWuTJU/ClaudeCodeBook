# Summary of `commands/bridge/`

## Purpose of `bridge/`

The `bridge/` directory implements the **Remote Control** feature — a bidirectional communication bridge between the CLI and claude.ai. It enables users to establish terminal sessions that can be controlled remotely via a web interface or mobile device, with support for both traditional (v1) and environment-less (v2) bridge modes.

## Contents Overview

| File | Role |
|------|------|
| `commands/bridge/index.ts` | **Command descriptor** — defines the `/remote-control` CLI command with lazy-loading, feature-flag gating, and metadata (name, aliases, hidden state) |
| `src/commands/bridge/index.js` | **Compiled implementation bundle** — contains the full React-based UI logic (Dialog, QR code rendering, keyboard navigation), state management, prerequisite checks, and analytics |

The TypeScript source file (`commands/`) acts as the entry point for the CLI. The compiled JavaScript bundle (`src/`) is the runtime implementation, containing:

- **`checkBridgePrerequisites()`** — validates org policy (`allow_remote_control`), version requirements (v1 via `checkBridgeMinVersion`, v2 via `checkEnvLessBridgeMinVersion`), and authentication tokens
- **`BridgeToggle`** — orchestrates the connection flow: checks prerequisites → enables bridge state → shows disconnect dialog if already connected → otherwise triggers remote callout
- **`BridgeDisconnectDialog`** — interactive dialog with three options: Disconnect, Show/Hide QR, Continue; includes cyclic keyboard navigation (↑/↓ to move focus, Enter to select, Esc to dismiss)
- **`useKeybindings`** — wraps key events with proper cleanup, used for dialog navigation
- **`qrcode`** integration — generates a colored terminal QR code (small size, low error correction) from the session URL for easy mobile scanning
- **Analytics** — logs `tengu_bridge_command` events for connect, disconnect, and preflight failures

## How Files Relate to Each Other

```
commands/bridge/index.ts          src/commands/bridge/index.js
┌────────────────────────────┐   ┌─────────────────────────────────────────┐
│ isEnabled()                │   │ checkBridgePrerequisites()              │
│  ├─ feature('BRIDGE_MODE') │   │  ├─ isPolicyAllowed('allow_remote...')  │
│  └─ isBridgeEnabled() ───────────►  ├─ checkBridgeMinVersion()            │
│                                    │  ├─ checkEnvLessBridgeMinVersion()   │
│ bridge object                     │  └─ getBridgeAccessToken()             │
│  ├─ name: "bridge"                │                                         │
│  ├─ aliases: ["remote-control",   │ BridgeToggle component                 │
│  │            "rc"]               │  ├─ reads: AppState                    │
│  ├─ load: () => import('./') ─────────►├─ sets: replBridgeEnabled           │
│  └─ immediate: true              │  └─ writes: showRemoteCallout           │
│                                    │                                         │
│                                    │ BridgeDisconnectDialog                 │
│                                    │  ├─ usesKeybindings()                  │
│                                    │  ├─ renders QR via qrcode library      │
│                                    │  └─ logs analytics                     │
└────────────────────────────────────┴─────────────────────────────────────────┘
```

The TypeScript file **gates access** to the feature; the bundled JavaScript file **provides the implementation**. Both share `isBridgeEnabled()` as a runtime dependency.

## Key Takeaways

1. **Feature-flag driven**: Bridge mode is entirely gated by `BRIDGE_MODE` (compile-time) and `isBridgeEnabled()` (runtime), making it safe to ship in binary form while controlling access via GrowthBook or environment configuration.

2. **Dual bridge versions**: The implementation branches between v1 (requires env vars) and v2 (env-less) based on `isEnvLessBridgeEnabled()` and assistant/KAIROS mode, with matching version checks in both the command descriptor and the prerequisite validator.

3. **Prerequisite cascade**: Before enabling bridge, the system checks (in order) org policy limits, minimum version requirements, and authentication — failing early with descriptive errors.

4. **Terminal-native UX**: The QR code dialog is rendered entirely in the terminal using Ink (`Box`/`Text`) components with ANSI-colored output, no external window required.

5. **Keyboard-first dialogs**: The disconnect dialog implements proper keyboard navigation with focus management, cleanup, and escape-to-dismiss, ensuring accessibility without a mouse.

6. **Lazy and immediate**: The command module is loaded lazily on first invocation (`load`), but executes immediately (`immediate: true`) once loaded — balancing startup cost and responsiveness.
