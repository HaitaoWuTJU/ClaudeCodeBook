# Summary of `tools/WebFetchTool/`

## Purpose of `WebFetchTool/`

Implements a secure web content fetching tool for Claude Code that retrieves URLs, converts HTML to markdown, and uses a secondary AI model to extract relevant information based on user prompts. The tool enforces domain blocklists, permission rules, and redirect security.

## Contents Overview

| File | Role |
|------|------|
| `preapproved.ts` | Security whitelist defining 80+ preapproved domains (with optional path restrictions) that the tool can access without prompting the user |
| `prompt.ts` | Tool configuration, descriptions, and prompt template builder that differentiates guidelines for preapproved vs. non-preapproved domains |
| `UI.tsx` | Terminal UI renderer using Ink, displaying fetch progress, URL summaries, and formatted results with HTTP status codes |
| `utils.ts` | Core implementation: URL validation, domain blocklist checking, HTTP fetching with redirect handling, HTML→Markdown conversion, LRU caching, and secondary model integration |
| `WebFetchTool.ts` | Main entry point: exports the tool definition, integrates all modules, implements permission checking, and orchestrates the fetch → convert → process pipeline |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                         WebFetchTool.ts                             │
│                  (Main tool definition + orchestration)             │
│                                    │                                │
│         ┌──────────────────────────┼──────────────────────────┐      │
│         │                          │                          │      │
│         ▼                          ▼                          ▼      │
│  ┌─────────────┐          ┌─────────────┐           ┌─────────────┐   │
│  │ preapproved │          │   prompt    │           │     UI      │   │
│  │   .ts       │          │   .ts       │           │   .tsx      │   │
│  │ (isPreappro │          │ (prompt     │           │ (render     │   │
│  │  -vedHost)  │          │  template)  │           │  functions) │   │
│  └──────┬──────┘          └──────┬──────┘           └─────────────┘   │
│         │                       │                                     │
│         │                       │                                     │
│         └───────────────────────┼─────────────────────────────────────┘
│                                 │
│                                 ▼
│                      ┌─────────────────┐
│                      │     utils.ts    │
│                      │  (core logic:   │
│                      │  fetch, cache,  │
│                      │  convert,       │
│                      │  validate)      │
│                      └────────┬────────┘
│                               │
└───────────────────────────────┼───────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │   External Services    │
                    │  (axios, turndown,     │
                    │   Anthropic API)       │
                    └───────────────────────┘
```

1. **`WebFetchTool.ts`** is the orchestrator—it imports and uses all other modules
2. **`utils.ts`** imports **`preapproved.ts`** (domain checks) and **`prompt.ts`** (prompt templates)
3. **`UI.tsx`** imports `WebFetchTool.ts` types to render tool results
4. **`preapproved.ts`** and **`prompt.ts`** are pure configuration with no dependencies on other files in this directory

## Key Takeaways

- **Multi-layered security**: Domain blocklists checked via Anthropic API before any network request; redirects restricted to same-origin (ignoring `www.` prefix); URLs validated for length and structure
- **Lazy loading for performance**: The `turndown` library (~1.4MB) is only imported when the first HTML fetch occurs, and response buffers are explicitly released before markdown conversion to allow garbage collection
- **Dual cache strategy**: `URL_CACHE` (50MB, 15-min TTL) stores fetched content; `DOMAIN_CHECK_CACHE` (128 entries, 5-min TTL) caches blocklist lookups to avoid redundant API calls
- **Deferred execution**: The tool runs after the initial LLM response, allowing for proper permission prompts and user confirmation before network access
- **Granular permission system**: Three rule types per hostname—`deny` (block unconditionally), `ask` (prompt user), `allow` (bypass prompts)—with preapproved domains as a baseline
- **Preapproved domain categories**: Covers major programming languages (Python, JS/TS, Rust, Go), frameworks (React, Django, FastAPI), databases (PostgreSQL, MongoDB, Redis), cloud platforms (AWS, GCP, Azure), and AI/ML tools (HuggingFace, Kaggle, PyTorch)—excluding GitHub general access for security
