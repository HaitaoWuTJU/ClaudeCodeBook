# Summary of `tools/ScheduleCronTool/`

## Purpose of `ScheduleCronTool/`

Provides a complete suite of tools for scheduling prompts to run automatically on cron expressions — the `/loop` skill in Claude Code. It covers the full lifecycle: creating scheduled jobs (one-shot or recurring), listing them, and cancelling them. Jobs can be session-only (lost on restart) or durable (persisted to disk).

## Contents Overview

| File | Role |
|------|------|
| `prompt.ts` | Prompts, descriptions, and feature gates (`isKairosCronEnabled`, `isDurableCronEnabled`) |
| `CronCreateTool.ts` | Implements the `/loop` command — validates cron syntax, enforces job limits, writes tasks to storage |
| `CronDeleteTool.ts` | Cancels a scheduled job by ID with ownership validation |
| `CronListTool.ts` | Lists all active jobs with human-readable schedules |
| `UI.tsx` | Ink/React rendering for CLI output of all three tools |

## How Files Relate to Each Other

```
prompt.ts (feature gates + prompt strings)
       │
       ├──▶ CronCreateTool.ts
       │         │
       │         ├──▶ reads  listAllCronTasks()        (cronTasks.js)
       │         ├──▶ writes addCronTask()             (cronTasks.js)
       │         ├──▶ activates setScheduledTasksEnabled()
       │         └──▶ uses buildCronCreatePrompt / buildCronCreateDescription
       │
       ├──▶ CronDeleteTool.ts
       │         │
       │         ├──▶ reads  listAllCronTasks()        (cronTasks.js)
       │         ├──▶ writes removeCronTasks()         (cronTasks.js)
       │         └──▶ uses buildCronDeletePrompt
       │
       └──▶ CronListTool.ts
                 │
                 ├──▶ reads  listAllCronTasks()        (cronTasks.js)
                 └──▶ uses buildCronListPrompt

All three tools ──▶ UI.tsx (render their messages)
```

- **`prompt.ts`** is the single source of truth for user-facing text and the feature-flag gates that guard all three tools.
- **`CronCreateTool.ts`** is the write path; **`CronListTool.ts`** and **`CronDeleteTool.ts`** are the read and delete paths — all three delegate actual storage I/O to `utils/cronTasks.js`.
- **`UI.tsx`** sits at the presentation boundary, formatting the structured output of each tool into CLI-renderable React/Ink components.

## Key Takeaways

1. **Two-tier gating** — `prompt.ts` gates the whole feature with `isKairosCronEnabled()` (build-time `AGENT_TRIGGERS` + GrowthBook runtime) and separately gates durable persistence with `isDurableCronEnabled()` (GrowthBook only). This lets you kill the feature fleet-wide without a redeploy.

2. **Limits enforced at creation time** — `CronCreateTool.ts` hard-blocks at 50 total jobs (`MAX_JOBS`) and prevents durable crons for teammates (no persistence across sessions would orphan them on restart).

3. **Jitter is built-in** — Recurring tasks avoid `:00`/`:30` minute marks to spread load; one-shot tasks on popular minutes can fire up to 90 seconds early. Prompts describe this automatically in the creation prompt.

4. **Teammate isolation** — `CronDeleteTool` and `CronListTool` filter by `agentId` so teammates see and manage only their own jobs.

5. **Deferred, read-only where possible** — All three tools set `shouldDefer: true`; list and delete additionally mark `isReadOnly: true` / `isConcurrencySafe: true` to aid scheduler scheduling and prevent accidental mutation.
