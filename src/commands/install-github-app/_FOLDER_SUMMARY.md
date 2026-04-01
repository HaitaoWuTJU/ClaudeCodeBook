# Summary of `commands/install-github-app/`

## Purpose of `install-github-app/`

The `install-github-app/` directory contains the interactive CLI workflow for installing Claude Code's GitHub App into a repository. It guides users through authenticating with GitHub, configuring API keys, and setting up GitHub Actions workflows that run Claude Code on pull requests.

## Contents Overview

This directory is a state-machine wizard composed of **8 React/Ink terminal UI steps** and **1 utility module**:

| File | Type | Role |
|------|------|------|
| `CheckGitHubStep.tsx` | UI Step | Verifies GitHub CLI (`gh`) is installed |
| `ConnectRepoStep.tsx` | UI Step | Authenticates user via OAuth or existing credentials |
| `ChooseRepoStep.tsx` | UI Step | Prompts user to select target repository |
| `CheckExistingSecretStep.tsx` | UI Step | Checks for existing `ANTHROPIC_API_KEY` secret |
| `ApiKeyStep.tsx` | UI Step | Collects new API key or selects existing/OAuth token |
| `WaitingForOAuthStep.tsx` | UI Step | Polls for OAuth completion with retry/error handling |
| `LinkSentStep.tsx` | UI Step | Confirms OAuth email/link sent to user |
| `WarningsStep.tsx` | UI Step | Displays pre-flight warnings before final setup |
| `SuccessStep.tsx` | UI Step | Reports successful installation and next steps |
| `setupGitHubActions.ts` | Utility | Creates workflow files, sets secrets, opens PR browser |
| `types.ts` | Types | Shared TypeScript interfaces (`Workflow`, `Warning`, `OAuthStatus`, `GitHubAppState`) |
| `index.ts` | Entry | Orchestrates the step state machine |
| `constants/github-app.ts` | Constants | Workflow templates, PR content, URLs |

## How Files Relate to Each Other

```
User runs 'claude code github install'
         │
         ▼
┌────────────────────────┐
│   index.ts (state machine)  │
│   manages current step │──────► loads step components
└───────────┬────────────┘
            │
            ▼
   ┌────────────────┐
   │ CheckGitHubStep│──gh --version──► proceed or exit
   └───────┬────────┘
           ▼
┌──────────────────────┐
│ ConnectRepoStep      │──► OAuthService.authenticateGitHub()
└───────┬──────────────┘         │
        │                       ▼
        │              ┌──────────────┐
        │              │LinkSentStep  │ (OAuth email sent)
        │              └──────┬───────┘
        │                       │
        │                       ▼
        │              ┌───────────────────┐
        │              │WaitingForOAuthStep│──poll──► OAuthStatus
        │              └─────────┬─────────┘
        │                        │
        │                        ▼ success
        ▼                        │
┌──────────────┐                 │
│ ChooseRepoStep│◄────────────────┘
└──────┬───────┘
       ▼
┌─────────────────────────┐
│ CheckExistingSecretStep │──checks existing secret──► useExistingSecret flag
└──────────┬──────────────┘
           ▼
┌──────────────────┐
│   ApiKeyStep     │──collects API key / OAuth token
└──────┬───────────┘
       ▼
┌──────────────┐
│WarningsStep  │──displays pre-flight issues
└──────┬───────┘
       ▼
┌──────────────┐
│SuccessStep   │──shows setup summary
└──────┬───────┘
       ▼
┌─────────────────────────────────────────────┐
│         setupGitHubActions.ts                │
│  • createWorkflowFile()                      │
│  • gh api repos/{repo}/actions/secrets/{name}│
│  • openBrowser(PR_URL)                       │
└─────────────────────────────────────────────┘
```

**Dependency chain:**
- All step components import shared UI primitives from `../../ink.js` (`Box`, `Text`, `color`, `useTheme`)
- All step components use `useKeybindings` from `../../keybindings/useKeybinding.js`
- `WaitingForOAuthStep` imports `OAuthService` from `services/oauth`
- `setupGitHubActions.ts` imports workflow templates from `constants/github-app.ts`
- `types.ts` defines the central `GitHubAppState` interface consumed by every step

## Key Takeaways

1. **Terminal-first UX**: Every UI component uses Ink (React for CLIs), not browser DOM. Text input, keyboard navigation, and terminal colors are handled via dedicated hooks and components.

2. **React Compiler output**: Most `.tsx` files are pre-compiled by React Compiler (`react/compiler-runtime`), evidenced by `_c` functions and Symbol-based memoization. Source maps are embedded for debugging.

3. **Flexible authentication**: The flow supports three credential modes — existing API key, newly entered API key, or full OAuth token — with automatic fallback based on what's already configured.

4. **GitHub CLI as backbone**: All GitHub API calls (repo listing, secret creation, workflow file creation, branch management) are delegated to the `gh` CLI via `execFileNoThrow`, not raw REST calls.

5. **OAuth polling with safety**: The `WaitingForOAuthStep` state machine handles waiting, success, error, retry, and cancellation with automatic cleanup via `useEffect` and token persistence via `saveOAuthTokensIfNeeded`.

6. **Workflow templating**: Two YAML workflow templates (`WORKFLOW_CONTENT`, `CODE_REVIEW_PLUGIN_WORKFLOW_CONTENT`) are stored as constants and substituted at setup time with the user's chosen secret name and workflow type.

7. **Analytics instrumentation**: Key lifecycle events (`tengu_setup_github_actions_started`, `tengu_setup_github_actions_completed`) are tracked via `logEvent` throughout the flow for telemetry.
