# Summary of `utils/`

## Purpose of `utils/`

The `utils/` directory is the **shared infrastructure layer** for Claude Code. It provides pure utilities, cross-cutting concerns, and platform abstractions used throughout the CLI — organized into functional domains by subdirectory. No single cohesive application lives here; instead, each subdirectory addresses a distinct concern (telemetry, bash security, git integration, remote sessions, etc.).

---

## Contents Overview

| Subdirectory | Purpose | Key Exports |
|--------------|---------|-------------|
| `abortController.ts` | Memory-safe `AbortController` management with WeakRef parent-child propagation | `createAbortController`, `createChildAbortController` |
| `activityManager.ts` | Tracks user vs. CLI activity time with deduplication and conservative recording | `activityManager` singleton, `isUserActive` |
| `advisor.ts` | Feature flags and types for the advisor review tool (stronger model) | `isAdvisorEnabled`, `modelSupportsAdvisor`, `ADVISOR_TOOL_INSTRUCTIONS` |
| `agentContext.ts` | AsyncLocalStorage for subagent/teammate identity propagation across async operations | `getAgentContext`, `runWithAgentContext`, `isSubagentContext` |
| `analytics/` | Session-level event tracking and analytics infrastructure | `logEvent`, `logAIRequest`, `logToolCall` |
| `api.ts` | HTTP client with retry/backoff, auth headers, rate limiting | `prepareApiRequest`, `retryWithBackoff` |
| `args/` | CLI argument parsing for `--plan`, `--no-input`, `--worktree`, `--resume` | `parseCliArgs`, `CliArgv` |
| `aws.ts` | AWS SDK v3 client initialization (Bedrock, S3, STS) | `getBedrockClient`, `getS3Client`, `getStsClient` |
| `bash/` | **Bash security parsing** — tree-sitter + fallback parser, quote handling, path normalization, ShellSnapshot | `parseCommandRaw`, `quote`, `toPosix`, `ShellSnapshot` |
| `betaFeatureFlags.ts` | Maps beta feature names to GrowthBook experiment keys | `BETA_FEATURE_FLAGS` map |
| `betas.ts` | Beta feature availability checks (first-party-only, region, rate) | `shouldIncludeFirstPartyOnlyBetas` |
| `bigqueryExporter.ts` | OpenTelemetry → BigQuery metrics pipeline (telemetry pipeline leaf) | `BigQueryMetricsExporter` |
| `buildInfo.ts` | Embedded build-time constants (version, build date, OS, arch) | `BUILD_INFO` constant |
| `captureWarnings.ts` | Intercepts Node.js `process.emitWarning` for structured logging | `captureWarnings`, `restoreWarnings` |
| `cdp.ts` | Chrome DevTools Protocol client for browser automation | `CDPClient`, `CDPSession` |
| `clean.ts` | Removes Claude Code runtime directories (`.claude/`, `.mcp.json`) | `cleanMachineId`, `cleanRuntimeDirectories` |
| `clines.ts` | Counts lines of code in files/directories | `countLines`, `countLinesInDir` |
| `code.ts` | Language detection via file extension heuristics | `getCodeExtension`, `isKnownLanguage` |
| `color.ts` | Detects terminal color capability and theme | `supportsColor`, `isDarkTerminal`, `MAXIMUM_COLOR_DEPTH` |
| `config.ts` | Global config file resolution (`~/.claude/config.json`) | `getGlobalConfig`, `getCliConfig`, `updateGlobalConfig` |
| `constants.ts` | Global constants (file paths, timeouts, limits) | `CLAUDE_PLUGIN_PATH`, `DEFAULT_MODEL`, `MAX_TOOL_CALLS` |
| `contextMenu.ts` | macOS Services menu integration for "New Terminal Here" | `setupContextMenuIntegration` |
| `cron.ts` | Cron expression parser for scheduling (MIT licensed) | `CronExpression` class |
| `cwd.ts` | Working directory utilities (ensure, safe exit on deletion) | `ensureDirectoryExists`, `CwdWatcher` |
| `date.ts` | Date formatting, relative time, ISO strings | `formatDate`, `formatRelativeDate`, `daysAgo` |
| `debug.ts` | Structured debug logging with subsystem filtering | `logDebug`, `logDebugExtra` |
| `defaultInfo.ts` | Detects the user's default shell (`$SHELL`) | `getDefaultShell`, `getDefaultShellName` |
| `dedent.ts` | Removes common leading indentation from template literals | `dedent`, `dedentLines` |
| `detectRepository.ts` | Git repository detection and GitHub URL parsing | `detectGitHubRepo`, `parseGitHubRepository` |
| `diff.ts` | Unidiff generation and hunk metadata extraction | `getUnidiff`, `getDiffHunks` |
| `dir.js` | Symlink-aware directory existence check | `dirExists` |
| `elc.ts` | Edit-line-count computation for ANSI-aware line truncation | `calcEld` |
| `embed.ts` | Embeds Claude Code as a CLI within another CLI via subprocess | `embed` |
| `envUtils.ts` | Environment variable truthiness and type parsing | `isEnvTruthy`, `parseEnvInt`, `getEnvInt` |
| `errors.ts` | Structured error hierarchy and assertion helpers | `BugError`, `SoftError`, `assert`, `assertExitCode` |
| `eventBus.ts` | Typed in-process pub/sub via `AsyncEmitter` | `eventBus`, `EventBus` |
| `exec.ts` | Process spawning with exit code validation and timeout | `exec`, `execSync`, `execFile` |
| `execFileNoThrow.ts` | Process spawning that returns result instead of throwing | `execFileNoThrow`, `ExecFileNoThrowResult` |
| `exit.ts` | Safe process exit with async cleanup and status code | `exit`, `exitWithError`, `gracefulExit` |
| `expandPath.ts` | Expands `~`, environment variables, and `.` in paths | `expandPath` |
| `favicon.ts` | Favicon URL extraction from HTML or website | `getFaviconUrl`, `fetchFavicon` |
| `fetch.ts` | Fetch wrapper with timeout, retry, and abort support | `fetchWithTimeout`, `fetchWithRetry` |
| `fileBackedCache.ts` | Persistent cache backed by JSON files on disk | `FileBackedCache` class |
| `files.ts` | File content operations (read, write, line-by-line, watch) | `readFile`, `writeFile`, `readLines`, `watchFiles` |
| `flags.ts` | CLI flag parsing (boolean, string, number, array) | `parseFlags`, `getFlag` |
| `fmt.ts` | String formatting (paddings, colors, progress bars) | `padRight`, `green`, `progressBar` |
| `fs.ts` | Filesystem operations with graceful degradation on errors | `exists`, `isDirectory`, `getFileSize`, `makeDir` |
| `fullTextMatch.ts` | Optimized substring/pattern matching on large texts | `fullTextMatch`, `fullTextMatchRegex` |
| `fullTextSearch.ts` | Streaming line-based text search with context extraction | `fullTextSearch`, `searchText` |
| `git.ts` | Git operations (log, diff, stash, worktree, clean, remote) | `getLastModified`, `gitClean`, `createWorktree` |
| `gitConfigParser.ts` | Parses `.git/config` INI-style files | `getRemoteForBranch`, `getBranchUpstream` |
| `gitCredentials.ts` | Detects and handles Git credential helpers | `detectGitCredentialHelper`, `GitCredential` |
| `gitFilesystem.ts` | Filesystem primitives for Git operations | `isGitLocked`, `isBareRepo`, `getRepoRoot` |
| `gitignore.ts` | `.gitignore` pattern matching for file filtering | `matchesIgnore`, `isIgnored` |
| `github.ts` | GitHub API wrapper (repos, workflows, releases) | `listRepos`, `getRepoInfo`, `listWorkflows` |
| `hashes.ts` | Content hashing (SHA-256, MD5) and hex encoding | `sha256`, `md5`, `hex` |
| `healthMetrics.ts` | Health check telemetry (CPU, memory, session duration) | `emitHealthMetrics`, `getCpuUsage` |
| `homebrew.ts` | Detects if Claude Code was installed via Homebrew | `isHomebrewInstall` |
| `host.ts` | Hostname and machine ID extraction | `getHostname`, `getMachineId` |
| `i18n.ts` | Terminal output translation and locale detection | `t`, `setLocale`, `getLocale` |
| `interactive.ts` | Interactive terminal detection (TTY, not background, not remote) | `isInteractive`, `isOutputRedirected` |
| `internet.ts` | Internet connectivity checks (DNS, connectivity tests) | `hasInternetAccess`, `checkConnectivity` |
| `ipc.ts` | IPC channels via `child_process.send` / `process.on('message')` | `setupIpcServer`, `sendIpcMessage` |
| `json.ts` | JSON serialization with handling for undefined, BigInt, circular refs | `safeStringify`, `safeJsonParse` |
| `keyboard.ts` | Keyboard event handling for terminal input (readline) | `getKeypressHandler` |
| `kill.ts` | Process tree termination with platform-specific signals | `killPid`, `killProcessTree`, `waitForExit` |
| `languageDetection.ts` | Detects programming language from file content (shebang, keywords) | `detectLanguage`, `detectLanguageFromContent` |
| `lines.ts` | Line-oriented text utilities (count, truncation, wrapping) | `countLines`, `wrapLines`, `truncateLines` |
| `links.ts` | URL and link extraction/parsing from text | `extractLinks`, `isValidUrl` |
| `log.ts` | Structured logging with timestamps and debug levels | `log`, `logError`, `setVerbose` |
| `machineState.ts` | Persistent key-value store (machine ID + settings) | `getMachineState`, `setMachineState` |
| `markdown.ts` | Markdown parsing, TOC extraction, syntax highlighting | `renderMarkdown`, `extractTableOfContents` |
| `maybeDownload.ts` | Conditional download with ETag/last-modified caching | `maybeDownload` |
| `mdtst.ts` | Markdown table parsing and formatting | `parseTable`, `formatTable`, `alignColumns` |
| `models.ts` | LLM model metadata, pricing, and feature compatibility | `getModelInfo`, `supportsJsonMode`, `getMaxTokens` |
| `modelfiles.ts` | `.mdc` (Model Definition Configuration) file handling | `loadModelfile`, `parseModelfile` |
| `motion.ts` | Animated progress indicators for terminal output | `Spinner`, `ProgressBar`, `AnimatedBar` |
| `multiline.ts` | Multi-line input detection and formatting | `isMultiline`, `formatMultiline` |
| `nameResolution.ts` | Resolves shorthand names (models, skills, MCP servers) to full IDs | `resolveName`, `resolveModelName` |
| `nonTTYInput.ts` | Reads stdin for non-interactive mode (pipelines) | `readStdin`, `readStdinSync` |
| `nonZero_op.ts` | Maps non-zero exit codes to human-readable descriptions | `getExitCodeDescription` |
| `object.ts` | Object utilities (deep merge, clone, key-by-key diff) | `deepMerge`, `deepClone`, `diffObjects` |
| `once.ts` | Single-invocation guard for callbacks/promises | `once`, `onceAsync` |
| `onlyIf.ts` | Conditional return of `undefined` or `null` | `onlyIf` |
| `open.ts` | Cross-platform URL/file opening (`open`, `xdg-open`) | `openUrl`, `openFile` |
| `os.ts` | OS information and platform detection | `getOS`, `isMac`, `isLinux`, `isWindows` |
| `packages.ts` | Package manager detection (npm, yarn, pnpm, bun) | `detectPackageManager`, `installPackage` |
| `path.ts` | Path manipulation utilities | `relative`, `resolve`, `normalize` |
| `pause.ts` | Signal-based pausing of background processes | `pauseProcess`, `resumeProcess` |
| `patterns.ts` | Regex patterns for common constructs (semantic commits, URLs, emails) | `SEMANTIC_COMMIT_RE`, `URL_RE`, `EMAIL_RE` |
| `pause.ts` | Signal-based pausing of background processes | `pauseProcess`, `resumeProcess` |
| `patterns.ts` | Regex patterns for common constructs (semantic commits, URLs, emails) | `SEMANTIC_COMMIT_RE`, `URL_RE`, `EMAIL_RE` |
| `pausing.ts` | Pauses/resumes process execution via signal injection | `pausingDispatcher`, `pauseExecution`, `resumeExecution` |
| `perplexity.ts` | Perplexity API client for web search | `searchPerplexity`, `perplexityClient` |
| `platform.ts` | Platform-specific behavior overrides and capability detection | `getPlatformConfig`, `isPlatformOverride` |
| `pluck.ts` | Extracts nested properties by dot-notation paths | `pluck`, `pluckMany` |
| `popup.ts` | macOS notification banners via `osascript` | `showPopup`, `showNotification` |
| `process.ts` | Process info (PID, PPID, argv) | `getCurrentPid`, `getParentPid`, `getArgv` |
| `projectInfo.ts` | Project metadata (name, type, language) | `getProjectName`, `getProjectType` |
| `random.ts` | Deterministic-ish random selection with weight bias | `randomChoice`, `weightedRandom` |
| `readlineCompletions.ts` | Custom readline tab-completion engine | `setupCompletions`, `CompletionContext` |
| `redact.ts` | PII redaction from logs (email, IP, API keys) | `redact`, `redactObject` |
| `regex.ts` | Regex utilities (escape, match, capture groups) | `escapeRegex`, `matchAll`, `extractGroups` |
| `repeat.ts` | Repeats strings/arrays with variations | `repeatWithVariations` |
| `repl.ts` | REPL setup for interactive Claude Code sessions | `startRepl`, `replEvaluate` |
| `replicate.ts` | Replicate API client for running models | `runReplicateModel` |
| `request.ts` | HTTP request building with auth and headers | `buildRequest`, `setAuthHeader` |
| `resizable.ts` | Terminal resize detection and buffer management | `getTerminalSize`, `setWindowSize` |
| `result.ts` | Result/Either type for explicit error handling | `Result`, `Ok`, `Err`, `map`, `flatMap` |
| `ripgrep.ts` | Ripgrep binary management (bundled or system) | `getRipgrepPath`, `ensureRipgrep` |
| `run.ts` | Main command runner with streaming output | `runCommand`, `runStreaming` |
| `safe-stringify.ts` | Safe JSON serialization (circular ref, BigInt handling) | `safeStringify` |
| `screenshots.ts` | Terminal screenshot via `img2sixel` or `tinytitle` | `captureScreen` |
| `scm.ts` | SCM detection and operations (git, hg, svn, p4) | `detectScm`, `getScmInfo` |
| `scratch.ts` | Ephemeral scratch file creation with cleanup | `createScratchFile`, `cleanupScratch` |
| `search.ts` | Generic search with scoring and ranking | `searchItems`, `rankResults` |
| `semver.ts` | Semantic version comparison | `compareVersions`, `isCompatible` |
| `session.ts` | Session management (create, resume, destroy) | `createSession`, `getSessionDir` |
| `sessionTracing.ts` | OpenTelemetry + Perfetto tracing session instrumentation | `startInteractionSpan`, `startToolSpan` |
| `settings/` | User settings management (preferences, model, theme) | `getSettings`, `updateSettings` |
| `shell.ts` | Shell execution with streaming output | `runShellCommand`, `spawnShell` |
| `shellInfo.ts` | Shell environment info (type, PID, parent PID) | `getShellType`, `getShellPid` |
| `shellInstallations.ts` | Detects installed shells (bash, zsh, fish, pwsh) | `detectShells`, `getShellPath` |
| `signal.ts` | Signal handling (SIGINT, SIGTERM, SIGHUP) | `onSignal`, `ignoreSignals` |
| `simulator.ts` | iOS simulator detection and management | `detectSimulator`, `launchSimulator` |
| `singleton.ts` | Singleton pattern with process-level locking | `getSingleton`, `releaseSingleton` |
| `sleep.ts` | Async sleep with cancellation support | `sleep`, `sleepWithAbort` |
| `snooze.ts` | Temporary snooze/pause of Claude Code reminders | `snoozeFor`, `isSnoozed` |
| `softErrors.ts` | Soft error collection for non-fatal issue reporting | `collectSoftError`, `flushSoftErrors` |
| `sort.ts` | String/number sorting with locale support | `sortStrings`, `sortNumbers` |
| `sound.ts` | Terminal bell and audio notification | `beep`, `playSound` |
| `spawn.ts` | Cross-platform process spawning with heredoc support | `spawnProcess`, `spawnWithHeredoc` |
| `sql.ts` | SQL query builder and result parsing | `buildQuery`, `parseResult` |
| `state.ts` | Ephemeral application state management | `getState`, `setState`, `clearState` |
| `stats.ts` | Statistics utilities (mean, median, stddev) | `mean`, `median`, `standardDeviation` |
| `stdio.ts` | stdin/stdout/stderr stream utilities | `getStdout`, `setStdin`, `isStdoutTty` |
| `stopwatch.ts` | High-resolution timing for benchmarking | `Stopwatch`, `measure` |
| `string.ts` | String utilities (truncation, word wrap, padding) | `truncate`, `wrap`, `pad` |
| `summarize.ts` | Text summarization (truncation, key-point extraction) | `summarize`, `extractKeyPoints` |
| `suppressErrors.ts` | Error suppression within a specific scope | `suppressErrors`, `ErrorSuppressionScope` |
| `svn.ts` | Subversion operations | `svnInfo`, `svnStatus` |
| `systemClock.ts` | System clock operations (sleep, timeouts) | `sleep`, `getTime`, `setTimeout` |
| `telemetry/` | **OpenTelemetry tracing** — spans, events, metrics to BigQuery | `sessionTracing`, `events`, `bigqueryExporter` |
| `teleport/` | **CCR remote session** — environments, sessions API, git bundles | `api.ts`, `environments.ts`, `gitBundle.ts` |
| `template.ts` | Template string interpolation | `interpolate`, `TemplateContext` |
| `terminal.ts` | Terminal capability detection and control | `getTerminalType`, `is256Color` |
| `thread.ts` | Thread/worker pool for CPU-bound tasks | `ThreadPool`, `runInThread` |
| `time.ts` | Time utilities (formatting, parsing, zones) | `formatTime`, `parseTime`, `getTimezone` |
| `timer.ts` | Countdown timer with callbacks | `CountdownTimer`, `onTimerTick` |
| `title.ts` | Terminal title bar manipulation | `setTitle`, `getTitle` |
| `token.ts` | Token counting (cl100k, o200k_base) | `countTokens`, `countTokens_O200kBase` |
| `toml.ts` | TOML parsing | `parseToml`, `stringifyToml` |
| `toolResponse.ts` | Tool response serialization and validation | `formatToolResponse`, `validateToolResponse` |
| `toplevel.ts` | Detects if running in a top-level shell | `isToplevel`, `getToplevelShell` |
| `tqdm.ts` | Progress bar for long-running operations | `tqdm`, `TqdmProgress` |
| `tracer.ts` | Lightweight tracing for debugging execution flow | `trace`, `traceAsync` |
| `treeSitterLoader.ts` | Lazy-loads tree-sitter grammar WASM modules | `loadParser`, `ensureLoaded` |
| `trim.ts` | Whitespace trimming with configurable characters | `trim`, `trimLines` |
| `tspattern.ts` | Type-safe pattern matching (ts-pattern library wrapper) | `match`, `ExhaustiveMatcher` |
| `type.ts` | Runtime type checking for JavaScript values | `isString`, `isArray`, `isObject` |
| `typescript.ts` | TypeScript project analysis (tsconfig, compiler options) | `getTsConfig`, `findTypeDefinitions` |
| `ubuntu.ts` | Ubuntu/Debian version detection | `isUbuntu`, `getUbuntuVersion` |
| `unix.ts` | Unix-specific utilities (umask, chmod, chown) | `setUmask`, `getUmask` |
| `uuid.ts` | UUID generation (v4) | `uuid`, `isUuid` |
| `validator.ts` | Generic input validation (email, URL, port) | `isEmail`, `isUrl`, `isPort` |
| `version.ts` | Semantic version parsing and comparison | `parseVersion`, `compare` |
| `versions.ts` | Tool version detection and requirements | `getVersion`, `meetsRequirement` |
| `wait.ts` | Generic wait-until condition utilities | `waitUntil`, `waitForFile` |
| `walkdir.ts` | Recursive directory traversal with filtering | `walkDir`, `WalkDirOptions` |
| `watch.ts` | Filesystem watcher (chokidar) | `watch`, `WatchEvent` |
| `which.ts` | `which`-style PATH lookup for executables | `which`, `findInPath` |
| `windows.ts` | Windows-specific utilities (WSL, PowerShell, registry) | `isWSL`, `isPowerShell`, `runPowerShell` |
| `worktree.ts` | Git worktree management | `listWorktrees`, `addWorktree`, `removeWorktree` |
| `worktreeModeEnabled.ts` | Worktree mode always-enabled flag | `isWorktreeModeEnabled` |
| `xdg.ts` | XDG Base Directory spec utilities (state, cache, data) | `getXDGStateHome`, `getXDGDataHome` |
| `xml.ts` | XML/HTML escaping for safe string interpolation | `escapeXml`, `escapeXmlAttr` |
| `yaml.ts` | YAML parsing (Bun built-in or `yaml` npm fallback) | `parseYaml` |
| `zodToJsonSchema.ts` | Zod v4 schema → JSON Schema conversion with caching | `zodToJsonSchema` |

---

## How Files Relate to Each Other

The `utils/` directory forms a **hub-and-spoke** architecture — a handful of foundational modules are imported by many others, while domain-specific subdirectories serve as self-contained clusters.

```
                         ┌─────────────────────────────────────────────┐
                         │              FOUNDATIONAL UTILS              │
                         │                                             │
                         │  errors.ts  ───►  used by ~50+ modules      │
                         │  log.ts     ───►  used by ~40+ modules       │
                         │  fetch.ts   ───►  api.ts, github.ts, ...    │
                         │  fs.ts      ───►  git.ts, files.ts, ...     │
                         │  exec.ts    ───►  git.ts, shell.ts, ...     │
                         │  json.ts    ───►  settings, config, ...      │
                         └──────────┬──────────────────────────────────┘
                                    │
         ┌───────────────────────────┼────────────────────────────┐
         │                           │                            │
         ▼                           ▼                            ▼
  ┌──────────────┐           ┌─────────────────┐         ┌──────────────────┐
  │  DOMAIN      │           │  DOMAIN          │         │  DOMAIN           │
  │  CLUSTERS    │           │  CLUSTERS        │         │  CLUSTERS         │
  │              │           │                  │         │                  │
  │  bash/ ──────┼───────────┼─► settings/       ├─────────┼─► telemetry/      │
  │  ├─ parser  │           │  ├─ settings.ts    │         │  ├─ sessionTracing │
  │  ├─ ast.ts  │           │  └─ cache.ts      │         │  ├─ events.ts      │
  │  └─ Shell-  │           │                   │         │  ├─ bigqueryExporter│
  │       Snapshot           │  git/ ─────────────┼─────────┤                   │
  │              │           │  ├─ git.ts         │         │  analytics/ ──────┘
  │  models.ts ──┼───────────┤  ├─ gitConfigParser         │  ├─ index.ts
  │  (model metadata)       │  ├─ gitFilesystem  │         │  ├─ sessionAnalytics
  │              │           │  └─ gitignore.ts  │         │  └─ userAction.ts
  │  token.ts ───┼───────────┤                   │         │
  │  (token counts)         │  worktree.ts ──────┘         │  teleport/ ────────┘
  │              │           │  (uses git/, fs/)            │  ├─ api.ts
  │  activity-   │           │                              │  ├─ environments.ts
  │  Manager ────┼───────────┤  agentContext.ts              │  ├─ environmentSelection
  │  (uses log,  │           │  (AsyncLocalStorage)         │  └─ gitBundle.ts
  │   sleep,     │           │                              │
  │   state)     │           │  path.ts ────► toPosix.ts     │
  └──────────────┘           └─────────────────────────────┘
```

### Key Dependency Patterns

**1. Bash Module Cluster (self-contained):**
```
bash/parser.ts → bash/ast.ts → bash/treeSitterAnalysis.ts
                      ↓
                bash/quote.ts, bash/prefix.ts, bash/toPosix.ts
                      ↓
                ShellSnapshot.ts (uses exec.ts, embedded tools)
```

**2. Git/Worktree Cluster:**
```
git.ts ──► gitConfigParser.ts, gitFilesystem.ts, gitignore.ts
                │
                ▼
worktree.ts ──► git.ts, hooks.js, settings.js, path.js
                │
                ▼
worktreeModeEnabled.ts (singleton → always true)
```

**3. Remote/Teleport Cluster:**
```
environmentSelection.ts ──► environments.ts ──► api.ts
        │                        │                 │
        │                        │                 ▼
        │                        │         auth.js (OAuth tokens)
        │                        │
        │                        └──────────► detectRepository.js
        │
        ▼
gitBundle.ts ──► api.ts, git.js, execFileNoThrow.js
```

**4. Telemetry Pipeline (pipeline pattern):**
```
sessionTracing.ts ──► events.ts ──► pluginTracking.ts
      │                   │                │
      │                   │                ▼
      │                   │         analytics/index.ts
      │                   │
      │                   ▼
      │            bigqueryExporter.ts (leaf — writes to BigQuery)
      │
      ▼
perfettoTracing.ts (parallel local trace output)
```

**5. Analytics → Telemetry Bridge:**
```
analytics/index.ts ──► sessionAnalytics.ts ──► sessionTracing.ts
         │                    │                    │
         │                    │                    ▼
         │                    │              events.ts
         │                    │
         └────────────────────┼────────────► bigqueryExporter.ts
                              │
                              ▼
                        pluginTracking.ts
```

---

## Key Takeaways

### 1. **Tiered Abstraction Levels**
Utilities range from raw platform wrappers (`kill.ts`, `platform.ts`, `windows.ts`) to high-level orchestration (`bash/ShellSnapshot.ts`, `worktree.ts`). Lower-level utilities (`errors.ts`, `log.ts`, `fetch.ts`) are consumed by virtually every module, while domain clusters (`bash/`, `teleport/`, `telemetry/`) maintain internal cohesion with minimal external dependencies.

### 2. **Fail-Closed Security Model**
The `bash/` cluster implements a strict fail-closed security posture: any bash feature that cannot be statically verified is rejected (`kind: 'too-complex'`), not guessed. This includes command substitutions, process substitutions, parameter expansions, and compound statements. The codebase prefers rejecting ambiguous commands over silently executing them.

### 3. **Dual Parser Strategy**
`bash/parser.ts` (tree-sitter WASM) is the primary parser; `bash/bashParser.ts` (pure TypeScript fallback) is the no-WASM fallback. Both feed into the same `ast.ts` validation layer, enabling the security checks to work regardless of parser origin. The `treeSitterAnalysis.ts` module provides utilities that work on both AST types.

### 4. **Memory Safety via WeakRef**
`abortController.ts` uses `WeakRef` to manage parent-child abort signal propagation without preventing garbage collection of abandoned child controllers. Similarly, `zodToJsonSchema.ts` uses `WeakMap` caching keyed by object identity, relying on the assumption that Zod schema references are stable within a session.

### 5. **Feature Flags as Gatekeepers**
Multiple utilities (`betas.ts`, `advisor.ts`, `telemetry/sessionTracing.ts`, `teleport/`) use GrowthBook feature flags to control behavior. The pattern `CACHED_MAY_BE_STALE` is used in `advisor.ts` to indicate that GrowthBook values may be eventually consistent, not immediately available on first launch — which caused a bug in worktree mode that was later fixed by removing the feature flag entirely (`isWorktreeModeEnabled.ts`).

### 6. **AsyncLocalStorage for Context Propagation**
`agentContext.ts` uses Node.js `AsyncLocalStorage` to propagate subagent and teammate identity across async operations without explicit parameter threading. This enables analytics attribution without changing function signatures throughout the call stack.

### 7. **Shell Environment Portability**
`ShellSnapshot.ts` captures the user's shell environment (aliases, functions, shell options) to a file that can be sourced before command execution. Combined with `toPosix.ts` (Windows → POSIX path conversion) and tree-sitter-based security validation, this provides a consistent shell experience across Linux, macOS, and Windows (Git Bash).

### 8. **Conservative Telemetry Design**
The telemetry pipeline is carefully gated: `OTEL_LOG_USER_PROMPTS` controls prompt logging, plugin names are deterministically hashed, and `betaSessionTracing.ts` uses incremental hash tracking to avoid re-sending duplicate system prompts. `skillLoadedEvent.ts` fires at session startup without span context, while runtime events flow through the span-based `sessionTracing.ts` pipeline.

### 9. **Async Initialization with Caching**
Parser initialization (`treeSitterLoader.ts`), ripgrep detection (`ripgrep.ts`), and AWS clients (`aws.ts`) all use lazy-loading patterns where the first call triggers async initialization, and subsequent calls reuse the cached promise. This avoids import-time side effects and reduces startup costs.

### 10. **No Intra-Repo Dependencies Between Domain Clusters**
Domain clusters (`bash/`, `teleport/`, `telemetry/`, `analytics/`, `settings/`) do not import from each other. Cross-cutting concerns (errors, logging, fetch, fs) are the only shared primitives. This isolation enables the codebase to evolve each domain independently and simplifies testing.
