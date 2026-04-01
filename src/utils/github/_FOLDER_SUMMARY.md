# Summary of `utils/github/`

## Purpose of `github/`

Provides GitHub CLI (`gh`) integration utilities for the repository, specifically for detecting the authentication state of the `gh` command-line tool used by the project's telemetry system.

## Contents Overview

| File | Purpose |
|------|---------|
| `ghAuthStatus.ts` | Checks if `gh` CLI is installed and authenticated |

## How Files Relate to Each Other

```
github/
└── ghAuthStatus.ts
```

A single-file module. Currently no inter-file dependencies within the directory — `ghAuthStatus.ts` is self-contained and only imports from sibling directories (`../which.js` and external `execa`).

## Key Takeaways

- **Purpose**: Detect `gh` auth state without making network requests — uses `gh auth token` which reads only local config/keyring
- **Security-conscious**: Token output is discarded (`stdout: 'ignore'`) so the actual token never enters this process's memory
- **Fast resolution**: 5-second timeout on auth check; `which()` lookup is O(n) PATH scan in Bun (no subprocess)
- **Graceful degradation**: Returns descriptive status strings instead of throwing errors

> **Note**: This directory contains only one file. A complete analysis of the `github/` subdirectory's scope would require examining other files if they exist in the repository.
