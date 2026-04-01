# Summary of `components/sandbox/`

## Purpose of `sandbox/`

The `sandbox/` directory contains React/Ink UI components for configuring and diagnosing Claude Code's sandbox execution environment. These components provide a terminal-based user interface for managing sandbox security settings, checking runtime dependencies, and displaying configuration status.

## Contents Overview

| File | Purpose |
|------|---------|
| `SandboxSettings.tsx` | Main tabbed configuration interface; orchestrates all sandbox-related tabs |
| `SandboxDependenciesTab.tsx` | Displays status of runtime dependencies (ripgrep, bubblewrap, socat, seccomp) with platform-specific install hints |
| `SandboxDoctorSection.tsx` | Diagnostic component that shows missing dependencies or warnings; renders only when issues exist |
| `SandboxOverridesTab.tsx` | Toggle UI for switching between "allow unsandboxed fallback" and "strict sandbox mode" |
| `SandboxConfigTab.tsx` | Read-only display of current sandbox configuration (filesystem rules, network restrictions, excluded commands, Unix sockets) |

## How Files Relate to Each Other

```
SandboxSettings.tsx (orchestrator)
│
├── SandboxModeTab (inline) ──────► SandboxManager ──► sandbox-adapter.js
│   └── Select dropdown
│
├── SandboxDependenciesTab.tsx ───► SandboxManager.checkDependencies()
│       └── Renders errors/warnings
│
├── SandboxOverridesTab.tsx ──────► SandboxManager.setSandboxSettings()
│   └── Select dropdown
│
├── SandboxConfigTab.tsx ─────────► SandboxManager (multiple getters)
│   └── Read-only display
│
└── SandboxDoctorSection.tsx ─────► SandboxManager.checkDependencies()
    └── Conditional rendering
```

**Data hierarchy:**
- `SandboxDoctorSection` and `SandboxDependenciesTab` both read from `SandboxManager.checkDependencies()` — the former shows aggregated status, the latter provides detailed breakdown
- `SandboxOverridesTab` and `SandboxConfigTab` both write/read from `SandboxManager` settings
- `SandboxSettings` acts as the container, conditionally rendering tabs based on dependency check results (errors block access to non-Dependencies tabs)

## Key Takeaways

1. **Platform-aware**: Components detect OS via `getPlatform()` and show Linux-specific warnings (bwrap/socat/seccomp) vs macOS (seatbelt)

2. **React Compiler optimization**: All components use `react/compiler-runtime` with manual cache slots (`$[0]`, `$[1]`, etc.) and `Symbol.for("react.memo_cache_sentinel")` for render optimization

3. **Dependency gating**: If sandbox dependencies have errors, users are blocked from configuring sandbox settings until dependencies are resolved

4. **Ink UI library**: All components use Ink (React for CLI) primitives — `Box`, `Text`, `Select`, `Link` — with theme support via `useTheme`

5. **Async state management**: Write operations (`setSandboxSettings`) are awaited before notifying parent via `onComplete` callbacks

6. **Conditional rendering patterns**: Heavy use of early returns (`react.early_return_sentinel`) and conditional tab visibility to minimize unnecessary UI mounting
