# Summary of `utils/deepLink/`

## Purpose of `deepLink/`

The `deepLink/` directory implements a complete protocol handler for launching Claude Code sessions from external sources via the `claude-cli://` URI scheme. When a user opens such a link (e.g., from a web page, Slack, or email), the OS invokes Claude Code with deep-link context, and this directory provides the infrastructure to parse the request, construct a security warning, resolve the working directory, and launch a new interactive terminal session.

## Contents Overview

| File | Role |
|------|------|
| `registerProtocol.ts` | OS-level registration of the `claude-cli://` URI scheme (macOS .app bundle, Linux .desktop file, Windows registry) |
| `parseDeepLink.ts` | Validates and parses raw `claude-cli://open?` URIs into structured `DeepLinkAction` objects with security checks |
| `protocolHandler.ts` | Trampoline entry point that receives the URI from the OS, coordinates resolution and launch |
| `terminalLauncher.ts` | Detects the user's preferred terminal emulator and spawns a new terminal running Claude with deep-link arguments |
| `terminalPreference.ts` | Captures the current `TERM_PROGRAM` and persists it to config (macOS-only), so the headless handler can use it later |
| `banner.ts` | Builds the multi-line security warning banner displayed when a deep-linked session starts |
| `index.ts` | Public re-exports for the module's public API |

## How Files Relate to Each Other

```
┌── OS invokes claude:// URL ──────────────────────────────────┐
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ registerProtocol.ts                                     │  │
│  │ Registers claude-cli:// scheme on macOS/Linux/Windows   │  │
│  │ so OS knows to launch Claude Code when link is clicked  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                    │
│                          ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ protocolHandler.ts  (handleUrlSchemeLaunch)              │  │
│  │ Receives raw URI from OS. Calls parseDeepLink,          │  │
│  │ resolves cwd, reads FETCH_HEAD, then calls              │  │
│  │ launchInTerminal from terminalLauncher.                  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                    │
│          ┌───────────────┼───────────────────────┐           │
│          ▼               ▼                       ▼           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │parseDeepLink │  │  banner.ts   │  │ terminalLauncher │    │
│  │              │  │              │  │                  │    │
│  │ - URI parse  │  │ - Build      │  │ - Detect term    │    │
│  │ - Validate   │  │   security   │  │ - Build CLI args │    │
│  │   (chars,    │  │   warning    │  │   --prefill,     │    │
│  │   length,    │  │   banner     │  │   --last-fetch,  │    │
│  │   slugs)     │  │ - Tildify    │  │   --deep-link-*  │    │
│  │              │  │   paths      │  │ - Spawn detached │    │
│  └──────────────┘  └──────────────┘  └──────────────────┘    │
│                                              │               │
│                                              ▼               │
│                               ┌──────────────────────────┐    │
│                               │ terminalPreference.ts   │    │
│                               │ Captures TERM_PROGRAM,   │    │
│                               │ persists for later use  │    │
│                               │ by headless handler     │    │
│                               └──────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Activation sequence:**

1. **Registration** — On first run (or upgrade), `registerProtocol.ts` writes OS-specific artifacts so the system recognizes `claude-cli://` links.

2. **User clicks a link** — The OS launches the Claude Code binary with `--handle-uri <url>`.

3. **Protocol handler receives** — `protocolHandler.ts` (invoked via `--handle-uri` flag) calls `parseDeepLink.ts` to extract `query`, `cwd`, and `repo` from the URI.

4. **Cwd resolution** — If no explicit `cwd` is given, `protocolHandler.ts` looks up the repo in `githubRepoPathMapping.js` (most-recently-used local clone).

5. **Banner construction** — `banner.ts` computes the repo's fetch age from `FETCH_HEAD` and formats a warning showing the working directory, prefill length, and staleness alert.

6. **Terminal detection** — `terminalLauncher.ts` detects the user's preferred terminal (checking the persisted `deepLinkTerminal` preference first on macOS, then environment/config fallbacks).

7. **Launch** — A new terminal window is spawned running the same Claude binary with `--deep-link-origin`, `--prefill`, `--deep-link-repo`, and `--deep-link-last-fetch` flags.

## Key Takeaways

- **Security is layered**: `parseDeepLink.ts` sanitizes Unicode and rejects control characters before any parameter reaches other code. `banner.ts` warns users to review pre-filled prompts, mitigating homoglyph and padding injection attacks.

- **Platform diversity**: The directory handles three distinct OS conventions (macOS LaunchServices bundle, Linux Freedesktop .desktop + xdg-mime, Windows registry) with platform-specific detection, quoting, and spawning strategies.

- **Terminal quoting is critical**: `terminalLauncher.ts` distinguishes pure-argv paths (terminal calls `execvp()` directly) from shell-string paths (terminal passes through a shell). For shell-string paths (iTerm2, Terminal.app, PowerShell, cmd.exe), correct quoting via `shellQuote()`, `appleScriptQuote()`, `psQuote()`, and `cmdQuote()` is load-bearing — broken quoting allows command injection.

- **The headless context problem**: When the OS launches Claude directly (no TTY, no terminal), environment variables like `TERM_PROGRAM` and config settings are unavailable. `terminalPreference.ts` solves this by persisting the terminal preference at startup for the headless handler to read later.

- **Graceful degradation**: Missing local clones, absent `FETCH_HEAD`, unrecognized terminals, and permission failures all degrade silently with logging rather than crashing the session.

- **Self-healing registration**: `isProtocolHandlerCurrent()` uses the symlink itself as a commit marker (written last among throwing calls). If the binary path changes or artifacts are deleted, the next session detects staleness and re-registers automatically.
