# Summary of `components/skills/`

## Summary of `skills/` Directory

### Purpose

The `skills/` directory contains the core infrastructure for loading, managing, and displaying Claude CLI skills in the terminal interface. It bridges the gap between filesystem-based skill definitions and the interactive CLI menu system.

### Contents Overview

| File | Purpose |
|------|---------|
| `loadSkillsDir.js` | Core module for discovering and loading skills from filesystem directories. Handles skill parsing, token estimation, and path resolution. |
| `SkillsMenu.jsx` (likely in a subdirectory) | React/Ink component rendering an interactive terminal dialog for browsing available skills. |

### How Files Relate to Each Other

```
loadSkillsDir.js          SkillsMenu.jsx
      │                         │
      │  ┌──────────────────────┘
      │  │   Imports functions:
      │  │   - getSkillsPath()
      │  │   - estimateSkillFrontmatterTokens()
      │  │
      │  │   Uses to:
      │  │   - Resolve skill file paths
      │  │   - Calculate token estimates
      │  │   - Determine display paths
      │  │
      ▼  ▼
┌─────────────┐
│  Skills UI  │
│  Workflow   │
└─────────────┘
```

- **`loadSkillsDir.js`** provides the data layer: scanning directories, parsing frontmatter, estimating resource usage, and resolving file paths
- **`SkillsMenu.jsx`** provides the presentation layer: filtering commands by type, grouping by source, and rendering an interactive terminal dialog
- The connection point is the skill commands array passed as props, with path/token utilities used for display purposes

### Key Takeaways

1. **Dual-layer architecture**: Separates skill discovery/loading from UI presentation
2. **Multiple skill sources**: Supports project settings, user settings, policy settings, plugins, and MCP servers
3. **Resource awareness**: Tracks token estimates for skills, enabling informed decisions
4. **Terminal-native UI**: Uses Ink library for rendering, making skills browsable in a non-graphical environment
5. **Metadata-rich display**: Shows source origins, server names, filesystem paths, and token costs
