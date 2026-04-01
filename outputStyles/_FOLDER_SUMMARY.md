# Summary of `outputStyles/`

## Purpose of `outputStyles/`

The `outputStyles/` directory contains the logic for loading and managing output style configurations in Claude Code. Output styles are customizable markdown-based templates that define how the AI should format its responses, allowing users and projects to define consistent output conventions.

## Contents Overview

| File | Description |
|------|-------------|
| `loadOutputStylesDir.ts` | Core module that loads markdown files from `output-styles` directories, parses frontmatter metadata, and transforms them into `OutputStyleConfig` objects |

## How Files Relate to Each Other

```
loadOutputStylesDir.ts
├── getOutputStyleDirStyles(cwd)
│   └── loadMarkdownFilesForSubdir('output-styles')
│       ├── Projects: .claude/output-styles/*.md
│       └── User: ~/.claude/output-styles/*.md
├── extractDescriptionFromMarkdown()  (from markdownConfigLoader)
├── coerceDescriptionToString()       (from frontmatterParser)
└── clearOutputStyleCaches()
    ├── Clears memoized getOutputStyleDirStyles cache
    ├── Clears markdown loader cache
    └── Clears plugin output style cache
```

The `loadOutputStylesDir.ts` acts as the central loader that:
1. Discovers markdown files from both project and user `output-styles` directories
2. Parses each file's frontmatter (name, description, `keepCodingInstructions`)
3. Extracts the file content as the prompt template
4. Returns standardized `OutputStyleConfig` objects for consumption by other parts of the codebase

## Key Takeaways

- **Two-tier loading**: Supports both project-level (`.claude/output-styles/`) and user-level (`~/.claude/output-styles/`) styles, with project styles taking precedence on name conflicts
- **Memoization**: Uses `lodash-es/memoize` to cache results, avoiding redundant filesystem reads
- **Frontmatter-driven configuration**: Styles are defined via markdown with YAML frontmatter for metadata
- **Flexible `keepCodingInstructions`**: Supports both boolean and string representations (`true`/`'true'`)
- **Error resilience**: Individual style parsing failures are isolated and don't prevent other styles from loading
- **Cache management**: Exposes `clearOutputStyleCaches()` for test isolation and runtime cache invalidation
