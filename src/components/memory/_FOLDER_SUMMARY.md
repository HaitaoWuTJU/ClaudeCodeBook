# Summary of `components/memory/`

# `memory/` Directory Summary

## Purpose of `memory/`

The `memory/` directory contains React components for managing Claude's memory functionality within a CLI interface built with **Ink** (React for terminals). It provides two primary features:

1. **File selection UI** — Allows users to browse, select, and open memory files (User memory, Project memory, auto-memory folders, and agent-specific memories)
2. **User notifications** — Displays contextual messages when memory files are updated

---

## Contents Overview

| File | Responsibility |
|------|----------------|
| `MemoryFileSelector.tsx` | Interactive selector for memory files with toggle controls for auto-memory and auto-dream features |
| `MemoryUpdateNotification.tsx` | Displays a notification banner showing which memory file was updated and how to edit it |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────┐
│                    Memory File Selection Flow                    │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
              ┌──────────────────────────────────┐
              │    MemoryFileSelector.tsx        │
              │                                  │
              │  • Reads memory files            │
              │  • Checks auto-memory/dream      │
              │    settings                      │
              │  • Renders toggle controls       │
              │  • Handles folder opening        │
              │  • Calls onSelect(path)          │
              └──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                  MemoryUpdateNotification.tsx                     │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
              Renders notification with formatted path
              "Memory updated in ~/CLAUDE.md · /memory to edit"
```

**Shared utilities** used by both:
- Path formatting: `getDisplayPath()`, `getRelativeMemoryPath()`
- Memory operations: `getMemoryFiles()`, `getAutoMemPath()`
- Configuration: `isAutoMemoryEnabled()`, `isAutoDreamEnabled()`
- Analytics: `logEvent()` for tracking user interactions

---

## Key Takeaways

| Aspect | Details |
|--------|---------|
| **Framework** | Ink (React for CLI) with React Compiler optimization (`_c` pattern) |
| **Features** | Nested memory file display, auto-memory/auto-dream toggles, folder opening, keyboard navigation |
| **Conditional Features** | Team memory (`feature('TEAMMEM')`), auto-memory folders (`feature('AUTO_MEM')`) |
| **Analytics** | Tracks `tengu_auto_memory_toggled`, `tengu_auto_dream_toggled` events |
| **Path Handling** | Supports User (`~/.claude/CLAUDE.md`), Project (`./CLAUDE.md`), and nested memory files |
| **Keyboard Support** | `confirm:yes`/`confirm:no` for selection, arrow keys for toggle cycling, Ctrl+C/D for exit |

The components are designed for a terminal-based UI and rely on sibling utilities in `utils/`, `services/`, and `memdir/` directories for file operations, settings persistence, and system integration.
