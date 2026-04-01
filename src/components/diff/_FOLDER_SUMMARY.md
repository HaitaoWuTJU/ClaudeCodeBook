# Summary of `components/diff/`

## Purpose of `diff/`

The `diff/` directory contains React components for rendering and navigating git diffs in a terminal UI built with Ink (React for CLIs). It provides a complete file-browser experience for viewing code changes, handling both list and detail views with keyboard navigation and multiple diff sources.

## Contents Overview

```
diff/
├── DiffDialog.tsx       # Main dialog orchestrator
├── DiffDetailView.tsx   # Single-file diff renderer with hunks
└── DiffFileList.tsx     # Paginated file list with selection
```

### DiffDialog — Orchestrator

The entry point and state manager for the diff experience:

- **Two view modes**: `'list'` (file browser) and `'detail'` (code hunks)
- **Multiple diff sources**: Current uncommitted changes + turn-based diffs extracted from conversation messages
- **Source navigation**: `◀` / `▶` toggles between sources when multiple exist
- **Keyboard shortcuts**: `↑`/`↓` navigate files, `Enter` opens detail, `Esc` closes or returns to list
- **State management**: Bounded `selectedIndex` and `sourceIndex`, auto-resets selection on source change
- **Empty states**: Loading, clean tree, no turn files, too many files

### DiffDetailView — File Detail Renderer

Displays the actual diff content for a single selected file:

- **File state handling**: Separate UIs for untracked, binary, large (>1MB), and normal files
- **Word-level diffing**: Delegates to `StructuredDiff` component for granular change highlighting
- **Terminal-responsive**: Width calculated as `columns - 2 * padding - 2 * border`
- **Caching**: `useMemo` for file content reads and first-line extraction
- **Truncation**: Shows "(truncated)" suffix and notice when diff exceeds 400 lines

### DiffFileList — File Browser

Renders the list of changed files with pagination:

- **Scroll window**: Max 5 visible files, centered on selection with edge-case handling
- **Path truncation**: `truncateStartToWidth()` preserves path start, truncates middle
- **Selection styling**: Bold + inverse highlighting for current file
- **File statistics**: Line counts (+/-) or status badges (untracked, binary, large)
- **Pagination indicators**: `↑ N more files` / `↓ N more files`

## How Files Relate to Each Other

```
DiffDialog (orchestrator)
├── useDiffData() ──────────► Current uncommitted git diff
├── useTurnDiffs() ─────────► Turn-based diffs from messages
│                              │
│                        sources[] array
│                              │
│         ┌────────────────────┴────────────────────┐
│         │                                         │
│  viewMode='list'                          viewMode='detail'
│         │                                         │
│         ▼                                         ▼
│  DiffFileList ◄── selectedIndex ──► DiffDetailView
│         │          (shared state)          │
│         │                                 │
│  ┌──────┴──────┐                   ┌──────┴──────┐
│  FileItem      FileStats           StructuredDiff (from ink)
```

**Data flow**:
1. `DiffDialog` fetches diff data from `useDiffData` (current) and `useTurnDiffs` (messages)
2. These are unified into a `sources[]` array with `DiffData` objects
3. `selectedIndex` picks a file from `diffData.files`
4. The selected file's `hunks[]` are passed to `DiffDetailView`
5. `DiffDetailView` reads actual file content and renders via `StructuredDiff`

## Key Takeaways

- **Composability**: Clean separation between dialog (state), file list (navigation), and detail view (rendering)
- **React Compiler**: All components use `_c()` cache pattern for optimized re-renders
- **Terminal-aware**: All layouts respect terminal dimensions via `useTerminalSize()`
- **Multiple sources**: The dialog bridges current git state and turn-based conversation history
- **Graceful degradation**: Handles binary files, large files, untracked files, and empty states with specific UI messaging
- **Accessibility**: Full keyboard navigation with shortcut hints displayed via `useShortcutDisplay()`
