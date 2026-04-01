# Summary of `hooks/`

## Purpose of `hooks/`

The `hooks/` directory is the **reactive orchestration layer** of the application. It contains React hooks, utility functions, and specialized subdirectories that bridge the gap between the user's terminal input, the AI agent's reasoning engine, and the external world (filesystem, git, MCP servers, voice services, permissions, and notifications). Every major user interaction—typing a prompt, pressing a voice activation key, approving a tool, or receiving feedback—flows through these hooks.

## Contents Overview

| File / Directory | Responsibility |
|-------------------|----------------|
| `fileSuggestions.ts` | Fuzzy file search using a Rust-based `FileIndex` with git/ripgrep fallbacks |
| `renderPlaceholder.ts` | Terminal placeholder text with cursor inversion and visibility control |
| `unifiedSuggestions.ts` | Combines file, MCP resource, and agent suggestions into a single ranked list |
| `useVoiceEnabled.ts` | Determines if voice is enabled (setting + auth + feature flag) |
| `useVoiceIntegration.tsx` | Integrates voice-to-text into the prompt input (hold-to-speak, cursor anchors) |
| `useVoiceInput.tsx` | WebSocket-based streaming STT with audio level visualization |
| `useVoiceState.tsx` | Global voice state management via React context |
| `useVoiceWebSocketManager.tsx` | WebSocket lifecycle management (connect, reconnect, teardown) |
| `useVoiceWebSocket.tsx` | Per-connection WebSocket hook with generation-based cleanup |
| `notifs/` | Hooks for domain-specific notifications (deprecation, limits, settings errors, etc.) |
| `toolPermission/` | Permission resolution for tool execution (hooks, classifier, user prompts) |
| `useMcpResources.tsx` | MCP resource URI fetching and metadata management |
| `usePromptAnalysis.tsx` | Parses prompt context flags (`@`, `#`, `/`, etc.) for special handling |
| `usePromptHistory.tsx` | Persistent prompt history with search and state management |
| `useQuickQuestion.tsx` | Handles inline quick-question modal with context preservation |
| `useRetry.tsx` | Retry logic with exponential backoff and jitter |
| `useToolCall.tsx` | Tool invocation with analytics, error handling, and permission pre-checks |
| `useToolResults.tsx` | Tool result rendering and result clearing |
| `useInput.js` | Raw keyboard input handling for terminal mode |
| `useTheme.tsx` | Theme access and subscription |

## How Files Relate to Each Other

The hooks form a layered architecture from low-level primitives to high-level UI integrations:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        UI / User Interaction                        │
│  PromptInput, ToolCallButton, QuickQuestion, VoiceIndicator         │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Prompt Input Integration Layer                    │
│                                                                       │
│  usePromptAnalysis ──────► useQuickQuestion                          │
│         │                        │                                   │
│  usePromptHistory ◄──────────────┘                                   │
│         │                                                             │
│  unifiedSuggestions ◄──── fileSuggestions ─── git ls-files / ripgrep │
│         │                     │                                       │
│         │            FileIndex (Rust)                                 │
│         │                                                             │
│  useVoiceIntegration ◄──── useVoiceEnabled                           │
│         │                        │                                    │
│  useVoiceInput ◄────────────── useVoiceState                         │
│         │                        │                                    │
│  useVoiceWebSocketManager ◄─── useVoiceWebSocket                     │
│         │                                                             │
│  WebSocket ──────────────────────────────────────────────────────────► STT API
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Notification System                           │
│                                                                       │
│  useDeprecationWarningNotification                                   │
│  useLimitsNotification                                               │
│  useSettingsErrors                                                   │
│  useTeammateShutdownNotification                                     │
│  useAutoModeUnavailableNotification                                 │
│  useCanSwitchToExistingSubscription                                  │
│                    ▲                                                 │
│                    │ useStartupNotification                          │
└────────────────────┼────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Permission Resolution System                      │
│                                                                       │
│  PermissionContext.js                                                │
│       │                                                             │
│       ├── handlers/coordinatorHandler ────► hooks + classifier      │
│       ├── handlers/interactiveHandler ────► dialog + bridge + hooks  │
│       └── handlers/swarmWorkerHandler ────► swarm leader delegation   │
│                                                                       │
│  permissionLogging.ts                                                │
│       │                                                             │
│       ├── Analytics (Statsig)                                        │
│       ├── OTel Telemetry                                             │
│       └── In-memory toolDecisions Map                                │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Tool Execution Layer                          │
│                                                                       │
│  useToolCall ──────► resolveToolPermission ──► Tool Execution        │
│       │                        │                                     │
│       └──► useToolResults ◄────┘                                     │
│                                                                       │
│  useMcpResources ──► MCP ServerResource fetching                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Cross-cutting concerns:**
- `useVoiceState.tsx` is the central state hub for all voice operations, providing both the state itself (`useVoiceState`) and a setter (`useSetVoiceState`) consumed by `useVoiceIntegration`, `useVoiceInput`, and `useVoiceWebSocketManager`.
- `useNotifications` context (from `src/context/notifications.js`) is consumed by every hook in `notifs/` and provides the `addNotification`/`removeNotification` API used throughout the codebase.
- `getIsRemoteMode()` is a universal kill-switch checked by every voice and notification hook to disable local-only features in remote/ssh mode.
- Feature flags (`VOICE_MODE`, `BASH_CLASSIFIER`, `TRANSCRIPT_CLASSIFIER`) gate entire subsystems, enabling dead-code elimination when disabled.

## Key Takeaways

1. **Reactive state is split across three contexts**: `useAppState` (global app settings and auth), `useVoiceState` (voice recording/transcript state), and `useNotifications` (notification queue). Each domain has its own isolated context to minimize re-renders.

2. **Performance is achieved through memoization, ref guards, and chunking**: Expensive operations like `hasVoiceAuth()` and file indexing are memoized. Race-condition guards (`lastSetInputRef`) prevent clobbering user edits. Async file operations yield to the event loop every ~10k files or 4ms.

3. **Feature flags drive compile-time and runtime behavior**: `feature('VOICE_MODE')` enables dead-code elimination of the entire voice subsystem. GrowthBook kill-switches (`isVoiceGrowthBookEnabled()`) allow mid-session toggling of features like the Bash classifier.

4. **Fallback chains provide robustness**: File suggestions fall back from `git ls-files` → ripgrep → directory-only results. Voice WebSocket reconnects with exponential backoff (max 30s). Permission handlers chain from coordinator → interactive mode.

5. **The permission system is a self-contained mini-framework**: `PermissionContext.js` provides the orchestration infrastructure, while the three handlers (`coordinator`, `interactive`, `swarmWorker`) implement different policies. `permissionLogging.ts` fans out to four telemetry destinations (analytics, OTel, code-edit metrics, in-memory map).

6. **Notifications are mostly one-shot**: Most notification hooks use `useRef` guards to fire at most once per session. `useLimitsNotification` and `useSettingsErrors` are notable exceptions—they remain live and re-evaluate when state changes.

7. **Voice activation uses input timing heuristics**: Hold-to-speak requires 5 rapid key events within 120ms (to distinguish from typing). Modifier combos activate on first press with a 2-second fallback timer to handle OS key-repeat delays.

8. **The file index uses progressive scoring**: Searches during background index build return partial results; a `indexBuildComplete` signal lets the typeahead UI re-run searches to upgrade from partial to complete results.

9. **Agent suggestions are Fuse.js filtered, file suggestions are nucleo-ranked**: The unified suggestions hook scores these two sources differently—pre-scored file results (0-1, lower = better) are merged with Fuse.js results (higher = better, inverted to match nucleo semantics).

10. **Swarm workers delegate permissions entirely to leaders**: `swarmWorkerHandler` has no user interaction path—it forwards all permission decisions to the swarm leader via mailbox, ensuring centralized policy enforcement.
