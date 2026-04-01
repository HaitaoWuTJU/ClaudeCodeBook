# Summary of `commands/`

## Purpose of `commands/`

The `commands/` directory is the central registry for all slash commands available in Claude Code. Each command is a self-contained module that defines a CLI action — ranging from configuration toggles (`/advisor`, `/brief`) to interactive tools (`/lens`, `/doctor`) and session management (`/login`, `/logout`). Commands follow a consistent interface (`Command` type) with optional lazy-loading to keep startup time minimal.

## Contents Overview

### Command Modules (Top-Level Files)

| Command | Type | Description |
|---------|------|-------------|
| `advisor.ts` | local | Configure advisor model |
| `bridge-kick.ts` | local | Inject bridge failures (Ant-only debug) |
| `brief.tsx` | local-jsx | Toggle brief-only mode |
| `browser-mcp.tsx` | local-jsx | Browser MCP tools setup |
| `claude-code-repl.tsx` | local-jsx | Interactive REPL |
| `connect-bridge.ts` | local | Bridge connection management |
| `create-mcp-tool.tsx` | local-jsx | Create custom MCP tools |
| `custom-commands.tsx` | local-jsx | Manage custom commands |
| `def.ts` | local | Execute custom commands |
| `developer.ts` | local | Toggle developer mode |
| `diagnostics.tsx` | local-jsx | Diagnostic information |
| `docs.tsx` | local-jsx | Documentation lookup |
| `doctor.ts` | local | Health checks |
| `env.ts` | local | Environment variable viewer |
| `file.tsx` | local-jsx | File content viewer |
| `git-diff.ts` | local | Git diff viewer |
| `git-log.ts` | local | Git history viewer |
| `git-status.ts` | local | Git status viewer |
| `guardrails.ts` | local | Security analysis |
| `help.tsx` | local-jsx | Help system |
| `install-mcp.tsx` | local-jsx | Install MCP servers |
| `install-plugin.tsx` | local-jsx | Install plugins |
| `lens.tsx` | local-jsx | Lens tool integration |
| `localization.tsx` | local-jsx | Localization viewer |
| `login.tsx` | local-jsx | Authentication |
| `logout.tsx` | local-jsx | Session termination |
| `mdk.ts` | local | MDK tool |
| `mdp.tsx` | local-jsx | MDP tool |
| `mermaid.tsx` | local-jsx | Mermaid diagram viewer |
| `mobile-preview.tsx` | local-jsx | Mobile preview |
| `model.ts` | local | Model selection |
| `mount.tsx` | local-jsx | Mount command |
| `nits.tsx` | local-jsx | Nits viewer |
| `package-search.tsx` | local-jsx | Package search |
| `patch.tsx` | local-jsx | Patch viewer |
| `permission.tsx` | local-jsx | Permission management |
| `permissions.tsx` | local-jsx | Permissions viewer |
| `plugins.tsx` | local-jsx | Plugin management |
| `preview.tsx` | local-jsx | Preview tool |
| `pr.tsx` | local-jsx | Pull request viewer |
| `project.tsx` | local-jsx | Project viewer |
| `prompt.tsx` | local-jsx | Prompt viewer |
| `read.tsx` | local-jsx | Read file content |
| `redo.tsx` | local-jsx | Redo action |
| `refresh.tsx` | local-jsx | Refresh command |
| `reject.tsx` | local-jsx | Reject action |
| `remote-session.tsx` | local-jsx | Remote session management |
| `repo.tsx` | local-jsx | Repository viewer |
| `revert.tsx` | local-jsx | Revert changes |
| `review.tsx` | local-jsx | Review tool |
| `role.tsx` | local-jsx | Role management |
| `root.tsx` | local-jsx | Root command |
| `rules.tsx` | local-jsx | Rules viewer |
| `sandbox.tsx` | local-jsx | Sandbox management |
| `search.tsx` | local-jsx | Search tool |
| `security.tsx` | local-jsx | Security analysis |
| `session.tsx` | local-jsx | Session management |
| `settings.tsx` | local-jsx | Settings management |
| `shell.tsx` | local-jsx | Shell integration |
| `skill.tsx` | local-jsx | Skill management |
| `skills.tsx` | local-jsx | Skills list |
| `snap.tsx` | local-jsx | Snapshot tool |
| `status.tsx` | local-jsx | Status viewer |
| `statusline.tsx` | prompt | Status line setup |
| `ultraplan.tsx` | local-jsx | Ultraplan tool |
| `version.ts` | local | Version info |

### Subdirectories

| Directory | Purpose |
|-----------|---------|
| `add-dir/` | Add workspace directory |
| `agents/` | Agent management |
| `ant-trace/` | Ant trace (stub) |
| `autofix-pr/` | Autofix PR (stub) |
| `backfill-sessions/` | Session backfill (stub) |
| `branch/` | Conversation branching |
| `bug-report/` | Bug reporting tool |
| `check/` | Code checking tool |
| `claude-code/` | Claude Code command |
| `code-review/` | Code review |
| `command-info/` | Command info |
| `config/` | Configuration viewer |
| `config-set/` | Set configuration values |
| `connect/` | Connection management |
| `context/` | Context management |
| `continue/` | Continue execution |
| `detach/` | Detach session |
| `diff/` | Diff viewer |
| `docs-search/` | Documentation search |
| `embeddings/` | Embeddings management |
| `feedback/` | Feedback submission |
| `file-diff/` | File diff viewer |
| `find/` | Find tool |
| `fix/` | Fix code issues |
| `fork/` | Fork session |
| `github-auth/` | GitHub OAuth |
| `github-browser/` | GitHub browser |
| `glob/` | Glob file search |
| `grep/` | Text search |
| `help-lens/` | Help lens |
| `install-skill/` | Install skills |
| `install-tool/` | Install tools |
| `integ-test/` | Integration testing |
| `invite/` | Invite users |
| `jira-auth/` | Jira OAuth |
| `jira-browser/` | Jira browser |
| `jira-connect/` | Jira connection |
| `jira-sync/` | Jira sync |
| `kill-session/` | Kill session |
| `kill/` | Kill command |
| `lab/` | Lab environment |
| `last/` | Last command |
| `log-stream/` | Log streaming |
| `logout/` | Logout session |
| `logs/` | View logs |
| `memory/` | Memory management |
| `mkdir/` | Make directory |
| `mobiles/` | Mobile sessions |
| `model-info/` | Model information |
| `mount/` | Mount viewer |
| `new/` | New session |
| `nvim/` | Neovim integration |
| `patch-check/` | Patch checking |
| `pause/` | Pause execution |
| `platforms/` | Platform management |
| `playwright/` | Playwright integration |
| `plugins/` | Plugin registry |
| `preview-config/` | Preview configuration |
| `projects/` | Projects management |
| `prompt-lens/` | Prompt lens |
| `providers/` | Provider management |
| `pull/` | Pull changes |
| `push/` | Push changes |
| `question/` | Question command |
| `read-line/` | Read line |
| `reauth/` | Re-authenticate |
| `remote/` | Remote sessions |
| `report/` | Report generation |
| `retry/` | Retry command |
| `review-check/` | Review checking |
| `run/` | Run command |
| `sandbox-access/` | Sandbox access |
| `sandbox-list/` | Sandbox listing |
| `scm/` | SCM integration |
| `search-skill/` | Search skills |
| `security-guardrails/` | Security guardrails |
| `security/` | Security viewer |
| `self-check/` | Self-check |
| `send/` | Send message |
| `settings/` | Settings viewer |
| `share/` | Share session |
| `shell-bridge/` | Shell bridge |
| `show-config/` | Show configuration |
| `skill/` | Skill viewer |
| `skills/` | Skills viewer |
| `slack/` | Slack integration |
| `snapshot/` | Snapshot viewer |
| `ssh/` | SSH integration |
| `start/` | Start session |
| `stats/` | Statistics |
| `stop/` | Stop command |
| `storybook/` | Storybook integration |
| `studio/` | Studio viewer |
| `suggest/` | Suggestions |
| `summarize/` | Summarization |
| `system-info/` | System information |
| `tail/` | Tail logs |
| `talk/` | Talk command |
| `tasks/` | Tasks viewer |
| `team/` | Team management |
| `terminal/` | Terminal viewer |
| `test/` | Test runner |
| `thinkback/` | Thinkback animation |
| `token/` | Token management |
| `tunnel/` | Tunnel management |
| `undo/` | Undo action |
| `unmount/` | Unmount viewer |
| `unpause/` | Unpause execution |
| `upgrade/` | Upgrade subscription |
| `usage/` | Usage viewer |
| `user-info/` | User information |
| `vim/` | Vim mode toggle |
| `voice/` | Voice dictation |
| `vscode/` | VS Code integration |
| `web/` | Web viewer |
| `which/` | Which command |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Command Registry                            │
│                    (commands.js — index of all commands)            │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
   ┌────────────┐   ┌────────────┐   ┌────────────────┐
   │ Type: local │   │Type: local-jsx│ │ Type: prompt   │
   │   (.ts)     │   │   (.tsx)      │ │                │
   └──────┬─────┘   └──────┬───────┘ └───────┬────────┘
          │                │                 │
          │         ┌───────┴───────┐         │
          │         │               │         │
          │         ▼               ▼         │
          │   ┌───────────┐  ┌─────────────┐   │
          │   │ Components│  │Components   │   │
          │   │  (UI)     │  │(UI) + Logic │   │
          │   └───────────┘  └─────────────┘   │
          │                                      │
          └──────────┬──────────────────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Shared Utilities  │
            │  • analytics/      │
            │  • state/          │
            │  • utils/          │
            │  • services/       │
            └────────────────────┘
```

**Entry Point Pattern** (most subdirectories):
```
subdir/
├── index.ts        → Command metadata + lazy load()
└── actual.tsx      → Implementation (loaded on demand)
```

**Pattern: Local command** (e.g., `advisor.ts`):
- Exports default object with `name`, `call()`, `isEnabled()`, etc.
- Synchronous or async execution

**Pattern: Local-JSX command** (e.g., `brief.tsx`):
- Uses `type: 'local-jsx'`
- `load()` function returns dynamic import of `.tsx` file
- Returns React components for interactive UIs

**Pattern: Prompt command** (e.g., `statusline.tsx`):
- `type: 'prompt'`
- `getPromptForCommand()` generates a prompt to be executed by a subagent

**Pattern: Stub** (e.g., `ant-trace/`, `backfill-sessions/`):
- `isEnabled: () => false`, `isHidden: true`
- Disabled/hidden features marked as unavailable

## Key Takeaways

1. **Uniform interface**: All commands satisfy the `Command` interface, enabling the CLI to discover, load, and execute them uniformly via the command registry.

2. **Lazy loading**: Most command modules use a `load()` function to defer heavy imports (React components, tool definitions), keeping startup fast.

3. **Multi-type architecture**: Commands exist in three execution modes:
   - **`local`**: Direct execution in the current process (`.ts`)
   - **`local-jsx`**: React-based interactive UIs rendered in the terminal (`.tsx`)
   - **`prompt`**: Generates a prompt for execution by a subagent

4. **Feature gating**: Commands use `isEnabled()` and `isHidden` flags, often driven by environment variables (`USER_TYPE`, feature flags) or GrowthBook experiments.

5. **Analytics integration**: Most stateful commands log events via `logEvent()` for telemetry and user behavior analysis.

6. **Tool integration**: Many commands leverage the shared tool system (`tools/`), re-using existing tool definitions rather than duplicating logic.

7. **Stub pattern**: Disabled features export minimal stubs (`isEnabled: () => false`, `isHidden: true`) to maintain code positions while marking features unavailable.

8. **Large surface area**: With 65+ top-level commands and 100+ subdirectories, the command system provides a comprehensive CLI experience covering authentication, file operations, git workflows, project management, security, and extensibility.
