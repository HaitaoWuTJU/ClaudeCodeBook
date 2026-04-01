# Summary of `utils/claudeInChrome/`

## Purpose of `claudeInChrome/`

This directory implements **Claude in Chrome** — a feature that integrates Claude Code with a Chrome browser extension, enabling Claude to interact with web pages, control the browser, and coordinate with MCP (Model Context Protocol) clients running Claude (such as Claude for Desktop or the Claude extension itself). It provides all the code needed for browser detection, extension installation, native messaging bridge setup, MCP server initialization, and terminal UI rendering for Chrome tools.

## Contents Overview

| File | Purpose |
|------|---------|
| `common.ts` | Shared utilities: browser configuration for 7 Chromium browsers, socket/named pipe path management, tab tracking (Set of tab IDs), URL opening via detected browser |
| `setupPortable.ts` | Portable extension detection scanning browser profile directories for installed extension IDs |
| `setup.ts` | Extension installation: creates native messaging manifests, wrapper scripts, Windows registry entries, and opens reconnect URL on first install |
| `prompt.ts` | Generates Claude-specific system prompt instructing Claude to use Claude-in-Chrome tools |
| `mcpServer.ts` | MCP server initialization: creates `ClaudeForChromeContext` with OAuth, analytics, socket bridge, and permission callbacks; registers `BROWSER_TOOLS` dynamically in MCP config |
| `chromeNativeHost.ts` | Chrome Native Messaging Host: binary (`--chrome-native-host`) that reads Chrome's stdin and forwards messages to MCP clients via Unix sockets/Windows named pipes |
| `toolRendering.tsx` | Custom terminal UI: renders tool usage messages, "View Tab" hyperlinks, and result summaries for Chrome MCP tools |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           EXTENSION DETECTION FLOW                           │
│                                                                              │
│  setupPortable.ts ──────► setup.ts ──────────► Native Messaging Manifests   │
│  (scans browser dirs)      (installs manifests,    + Wrapper Scripts        │
│  ↓                          wrapper scripts,        + Registry Entries       │
│  common.ts                  registry entries)                              │
│  (CHROMIUM_BROWSERS,                                                    │
│   BROWSER_DETECTION_ORDER)                                                  │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                            RUNTIME EXECUTION FLOW                             │
│                                                                              │
│  ┌──────────────────────┐                                                    │
│  │   Chrome Extension    │                                                    │
│  │   (in Chrome browser) │◄──── native messaging (stdin/stdout)               │
│  └──────────────────────┘                                                    │
│              │                                                               │
│              ▼                                                               │
│  ┌─────────────────────────┐    socket    ┌──────────────────────────────┐   │
│  │   chromeNativeHost.ts   │◄────────────►│   mcpServer.ts                │   │
│  │   (--chrome-native-host │    bridge    │   (ClaudeForChromeContext)    │   │
│  │    binary mode)         │              │   • OAuth callbacks          │   │
│  └─────────────────────────┘              │   • Analytics logging        │   │
│          ↑                                │   • Permission handling      │   │
│          │                                │   • MCP server registration  │   │
│          │                                └──────────────┬───────────────┘   │
│          │                                               │                   │
│          │                    common.ts ◄────────────────┘                   │
│          │                    (socket paths, browser detection,            │
│          │                     tab tracking, openInChrome)                  │
│          │                              ▲                                   │
│          │                              │                                   │
│          │                    toolRendering.tsx                             │
│          └──────────────────────────────┘                                   │
│                         (MCP tool UI overrides)                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Installation-time** → `setupPortable.ts` detects if extension is installed; `setup.ts` creates native messaging manifests and wrapper scripts.

**Runtime** → `mcpServer.ts` registers Claude-in-Chrome tools with the MCP server. When the user runs `claude --chrome`, Claude Code reads the MCP config and the MCP server runs inside Claude Code's process. When Claude uses a Chrome tool, the MCP server forwards it through the socket bridge to `chromeNativeHost.ts`. The native host runs as a separate binary process communicating with Chrome via stdin/stdout using Chrome's native messaging protocol (4-byte length prefix). Responses flow back through the same path. `toolRendering.tsx` provides custom terminal UI rendering for all Chrome tool interactions.

## Key Takeaways

1. **Dual bridge architecture**: Communication between Chrome and Claude Code uses two mechanisms — Unix/Windows sockets for MCP ↔ native host communication, and Chrome's native messaging protocol (length-prefixed binary) for native host ↔ Chrome extension communication.

2. **Browser-agnostic**: Supports 7 Chromium-based browsers (Chrome, Brave, Arc, Chromium, Edge, Vivaldi, Opera) across macOS, Linux, and Windows, with platform-specific paths and permissions handling.

3. **Security-conscious**: Socket files use `0o600/0o700` permissions, usernames are included in socket names to namespace multi-user systems, and Windows registry entries are scoped per-browser with ACL verification.

4. **Native build distinction**: In the bundled CLI binary, `chromeNativeHost.ts` runs as `--chrome-native-host` subprocess. In development (TUI), it runs as a CLI script (`cli.js --chrome-native-host`).

5. **Caching trade-off**: Extension installation detection caches only positive results to avoid false negatives in shared config environments, accepting that dev environments may show incorrect "already installed" states.

6. **OAuth integration**: `mcpServer.ts` handles the full OAuth flow (`getChromeExtensionAuthTokens`, `saveChromeExtensionAuthTokens`) bridging the extension's browser-based OAuth with CLI's token management.

7. **Tab coordination**: A `Set<number>` (max 200 entries) tracks active tab IDs, enabling the terminal UI to render clickable `[View Tab]` links and supporting analytics/cleanup of tab state.

8. **Wrapper script requirement**: Chrome's native messaging manifest requires the `path` field to be an executable without arguments, necessitating wrapper scripts (`.bat` on Windows, shell on Unix) to pass the actual command with arguments.
