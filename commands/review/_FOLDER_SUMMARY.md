# Summary of `commands/review/`

## Purpose of `review/`

Implements the **ultrareview command** — a teleported, remote-executed code review session for Claude. This subdirectory handles feature gating, billing eligibility, user confirmation dialogs, and the full launch pipeline for running AI-powered reviews on pull requests or local branches.

## Contents Overview

| File | Role |
|------|------|
| `ultrareviewEnabled.ts` | Feature flag gate — hides the command from users/organizations that aren't enabled |
| `reviewRemote.ts` | Core logic — billing gate evaluation, session creation, teleportation, and remote task registration |
| `UltrareviewOverageDialog.tsx` | React dialog — asks users to confirm pay-per-use billing when free quota is exhausted |
| `ultrareviewCommand.tsx` | Command handler — orchestrates the feature gate, billing dialog, and launch call |

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────┐
│                      Command Entry                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ ultrareviewEnabled.ts                                        │
│  isUltrareviewEnabled()  ──►  filters command from CLI if   │
│     (GrowthBook flag)        feature flag is false           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ ultrareviewCommand.tsx                                       │
│                                                              │
│  call() {                                                    │
│    1. checkOverageGate()     ──► reviewRemote.ts             │
│    2. if needs-confirm:      ──► renders UltrareviewOverage  │
│         <OverageDialog>            Dialog.tsx                │
│    3. if proceed:                                            │
│         launchRemoteReview()   ──► reviewRemote.ts           │
│    4. confirmOverage()        ──► reviewRemote.ts             │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ reviewRemote.ts                                               │
│                                                              │
│  sessionOverageConfirmed  ← module-level flag                │
│  confirmOverage()          ← sets session flag               │
│  checkOverageGate()        ← quota + utilization fetch       │
│  launchRemoteReview()      ← teleport + task registration    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ UltrareviewOverageDialog.tsx                                 │
│                                                              │
│  <Dialog>                                                    │
│    "Your free reviews are exhausted"                         │
│    <Select options={options} onChange={handleSelect} />      │
│    "This review bills as Extra Usage"                        │
│                                                              │
│  handleSelect(value):                                         │
│    "proceed"  → calls onProceed(signal)  → parent launches   │
│    "cancel"   → calls handleCancel()      → parent aborts    │
│                                                              │
│  handleCancel():                                             │
│    abortControllerRef.current.abort()                        │
│    onCancel()                                                │
└─────────────────────────────────────────────────────────────┘
```

### Dependency Graph

```
ultrareviewCommand.tsx
├─ imports: reviewRemote.ts         (checkOverageGate, confirmOverage, launchRemoteReview)
├─ imports: UltrareviewOverageDialog.tsx
└─ imports: types/command.js

UltrareviewOverageDialog.tsx
└─ imports: CustomSelect, Dialog, Box, Text (design system)

reviewRemote.ts
├─ imports: analytics/growthbook.js     (GrowthBook feature config)
├─ imports: services/api/ultrareviewQuota.js, usage.js
├─ imports: tasks/RemoteAgentTask/
├─ imports: utils/auth.js, git.js, detectRepository.js, teleport.js

ultrareviewEnabled.ts
└─ imports: services/analytics/growthbook.js (cached feature flag read)
```

## Key Takeaways

### Two-Layer Feature Gating

1. **Compile-time gate** (`ultrareviewEnabled.ts`): Controls whether the `/ultrareview` command appears in the CLI at all — controlled by GrowthBook's `tengu_review_bughunter_config.enabled` flag.
2. **Runtime billing gate** (`checkOverageGate`): After the command is invoked, checks quota utilization and account type to determine whether to proceed, show a dialog, or block the action.

### Billing States Are a Discriminated Union

The `OverageGate` type in `reviewRemote.ts` (`proceed` | `not-enabled` | `low-balance` | `needs-confirm`) is exhaustively matched in `ultrareviewCommand.tsx`, ensuring every new billing state must be handled at compile time.

### Session-Level Confirmation Persistence

`sessionOverageConfirmed` in `reviewRemote.ts` is a **module-level flag** (not per-call). Once a user confirms Extra Usage billing during any `/ultrareview` call, all subsequent calls in the same session skip the confirmation dialog automatically.

### AbortController for Graceful Cancellation

`UltrareviewOverageDialog` creates an `AbortController` and passes its signal to `onProceed`. If the user presses Escape or closes the dialog before launch completes, the signal is aborted and `ultrareviewCommand.tsx` skips calling `confirmOverage()` and `onDone()`, preventing orphaned sessions from billing.

### Dual Execution Modes

`launchRemoteReview` decides at runtime whether to run in **PR mode** (`--pr N`) or **branch mode** (bundled working tree + `git merge-base` diff), based on whether the user passed a PR number to the command.

### Synthetic Environment for Billing

`CODE_REVIEW_ENV_ID = 'env_011111111111111111111113'` allows billing to route through org-service keys without requiring per-org CCR environment setup — a simplification for the launch path.

### GrowthBook-Controlled Remote Worker Config

Remote reviewer fleet size, per-task duration, and agent timeouts are all derived from the GrowthBook feature flag `tengu_review_bughunter_config`, with conservative hard-coded defaults (e.g., 22 min total wallclock to stay under the 30 min poll timeout with headroom for finalization).
