# Summary of `utils/computerUse/`

## Purpose of `utils/computerUse/`

Implements the **Claude Code computer-use (CU) surface** — a macOS-only capability that lets Claude control the user's desktop by driving a hidden Safari tab via the `@ant/computer-use-mcp` server. It orchestrates screenshot capture, app filtering, permission dialogs, file-based locking, and turn-end cleanup.

## Contents Overview

| File | Role |
|------|------|
| `wrapper.tsx` | **Entry point / top-level adapter.** Bridges `ToolUseContext` → `bindSessionContext` (MCP package). Caches the binding, dispatches calls, runs permission dialogs, transforms image blobs to API format. |
| `executor.ts` | **Native executor.** Thin wrapper around `swiftLoader.ts` that exposes `captureExcluding`, `captureRegion`, `listInstalledApps`, and `resolvePrepareCapture` — all requiring `CFRunLoop` pumping. |
| `swiftLoader.ts` | **Native module loader.** Lazy singleton loader for `@ant/computer-use-swift` with a `darwin`-only platform check. |
| `screenshots.ts` | **Screenshot pipeline.** Orchestrates capture (executor → JPEGs), display resolution (model-pinned → auto-resolve), filtering (sends only delta pixels), and multi-display support. |
| `computerUseLock.ts` | **File-based lock.** Atomic O_EXCL lock at `~/.claude/computer-use.lock` with stale-PID recovery. Coordinates single CU session per machine. |
| `hostAdapter.ts` | **Host API bridge.** Maps `com.anthropic.claude-code.HostAdapter` calls → MCP `CallTool` requests on stdin, streaming back responses. |
| `appNames.ts` | **Noise filtering + injection hardening.** Strips Spotlight noise (daemons, helpers, XPC services) and sanitizes app names (40-char cap, Unicode allowlist, dedup). |
| `common.ts` | **Shared utilities.** Terminal bundle ID detection (`__CFBundleIdentifier` + fallback map) and `CLI_CU_CAPABILITIES` static config. |
| `toolRendering.tsx` | **Terminal UI rendering.** Transforms CU tool inputs into user-facing strings (coordinates, truncated text, scroll directions) and one-line result summaries for Ink/React. |
| `cleanup.ts` | **Turn-end cleanup.** Un-hides apps hidden during the turn, releases the file lock, unregisters the Esc hotkey, and sends an exit notification. |
| `gates.ts` | **Feature gate.** Single flag `tengu_malort_pedway` controlling all CU logic paths. |

## How Files Relate to Each Other

```
wrapper.tsx  (public API surface)
    │
    ├─► executor.ts
    │       └─► swiftLoader.ts ──► @ant/computer-use-swift (native)
    │
    ├─► screenshots.ts
    │       └─► executor.ts (capture)
    │       └─► appNames.ts (name filtering)
    │       └─► common.ts (bundle ID / capabilities)
    │
    ├─► hostAdapter.ts ──► MCP server stdin/stdout
    │
    ├─► computerUseLock.ts (acquire/release/check)
    │
    ├─► toolRendering.tsx (display formatting)
    │
    ├─► cleanup.ts
    │       ├─► executor.ts (unhide)
    │       ├─► computerUseLock.ts (release)
    │       └─► escHotkey.ts (unregister)
    │
    └─► gates.ts (feature flag guard)
```

## Key Takeaways

1. **macOS only.** The native Swift module, O_EXCL lock on `~/.claude/`, and `__CFBundleIdentifier` lookup are all Darwin-specific. All non-`darwin` paths are guarded or absent.

2. **Lock is the coordination primitive.** A single file-based lock ensures only one CU session runs at a time, with stale-PID recovery to handle crashes. All other concurrency concerns (unhide, Esc hotkey, notification) are secondary.

3. **Screenshot pipeline is a filter → transform → format chain:**
   - **Capture** → JPEGs from the native module
   - **Filter** → Compare against previous frame; send only changed pixels (with `displayId` and `dimensions` metadata)
   - **Display resolution** → Model can pin to a display; auto-resolve cycles through displays if apps aren't in the pinned one
   - **Format** → MCP image blocks → Anthropic base64 image objects

4. **Injection hardening lives in `appNames.ts`.** App names are attacker-controlled (from Spotlight). The blocklist strips helper/daemon/service noise; the allowlist (Unicode-aware `\p{L}\p{M}\p{N}`) prevents multi-line injection; the 40-char cap prevents buffer-length tricks.

5. **The binding is cached process-wide.** `wrapper.ts` builds `bindSessionContext` once and reuses the dispatcher closure, so the screenshot blob persists across calls without re-requiring the native module.

6. **Permission dialogs are async Promises bridged to `setToolJSX`.** `wrapper.ts` uses `addEventListener('abort', ...)` and a shared ref to propagate Ctrl+C to the dialog, ensuring the lock is released even on user abort.

7. **`CFRunLoop` pumping is required** for four `@MainActor` Swift methods (`captureExcluding`, `captureRegion`, `apps.listInstalled`, `resolvePrepareCapture`). The comment in `swiftLoader.ts` flags this as a known requirement.
