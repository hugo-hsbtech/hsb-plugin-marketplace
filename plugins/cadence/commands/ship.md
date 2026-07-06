---
description: Autonomously execute a parallel cycle plan — flows end to end (no merge gates), per task drives worktree→branch→PR (stacked on its blocker, or integration) then monitors comments/CI/conflicts until a human merges. Never pushes to main; never merges unless the user explicitly authorizes it.
argument-hint: "<path to a cycle-plan .md (docs/plans/proposed/<datetime>-<slug>-<task-id>.md, or leave empty to use the plan in context / run /cadence:plan first)>"
---

Load and follow the `cadence-executor` skill (plugin `cadence`) to autonomously execute
a parallel cycle plan. This **implements and drives PRs to merge-ready and FLOWS end
to end** — it never freezes a task waiting for another's PR to merge (dependencies are
expressed by PR base: stacked on a single blocker, or targeting integration for 0/2+
blockers). Hard rules: **never push or commit to `main`; by default never merge a PR —
the human merges — unless the user explicitly authorizes merging for a named
task/wave/cycle; keep any not-yet-mergeable PR a draft; and keep monitoring each PR
while it is open.**

## Plan input

```
$ARGUMENTS
```

Resolve the plan:
- **A file path** → read that cycle plan (e.g.
  `docs/plans/proposed/<YYYYMMDD-HHMM>-<slug-of-proposed>-<task-id>.md`); take the
  canonical `slug` from the plan's metadata header, not the filename.
- **Empty, but a wave schedule is already in context** → use it.
- **Empty with no plan** → run `/cadence:plan` first (or ask the user for tasks).
  Do not invent tasks.

## What to do

Follow the `cadence-executor` skill end to end. **You run as a thin top orchestrator
that delegates each task to its own per-task orchestrator agent — you don't
implement, monitor, fix, or write task state yourself.**
1. Locate-or-create the run **state directory**
   `.cadence/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/` (`run.json` +
   `tasks/<id>.json`; glob `*-<slug>-cycle/run.json` + matching `planPath` to resume;
   per `references/execution-state.md`). Open the integration branch + plan PR.
2. **Each tick, spawn one `Agent` per IDLE active task** (no agent in flight) in a
   single message — `pending`→spec, `specified`→implement, `open`→monitor,
   `merged`→cleanup; **skip any task whose agent is still running**
   (`specifying`/`implementing`/`fixing`) so you never tick a PR mid-round-trip
   (idle-gating). **Pick each agent's model by phase:** spec/analysis → **Opus, high
   effort** (always); implement → by the complexity the spec found (high → Opus/medium,
   medium/low → Sonnet); monitor/cleanup → Sonnet. Don't run everything on Opus, and
   don't run analysis on a cheap model. Spec/implement/fix agents run in the background.
   Each agent
   resumes from its `tasks/<id>.json`, owns a **durable git worktree** with a
   **descriptive** `cadence/<slug>-t<id>-<task-slug>` branch (the slug reflects what the
   task does) off its **base** — the integration branch (0/2+ blockers) or its **single
   blocker's branch** (stacked). It advances its task one
   step: spec (superpowers brainstorming→writing-plans, decides complexity) → implement
   (TDD, auto-approving gates) → open its PR → (later, when settled) monitor/fix →
   cleanup. It is the **sole writer of its `tasks/<id>.json`** and does all its own
   `gh` work, and it **dies when its PR
   merges**.
3. **Flow, don't gate on merges.** A task is dispatched as soon as its **base branch
   exists** — a stacked child the moment its blocker's branch is pushed, not when it
   merges. Each task PR targets its base (integration, or its blocker's branch), never
   `main`; keep any PR that isn't safe to merge (stacked / CI-not-green / agent
   in-flight) as a **draft**. PR bodies follow `references/pr-template.md`, **sized to
   complexity** (a simple low-complexity change gets a short body, not the full
   mermaid+UAT template), didactic, referencing siblings by **PR number**, not bare
   task ids.
4. The per-task agent monitors its own PR every tick: **judge** each review/comment on
   its merits (agree → fix · better alternative → fix differently · wrong/out-of-scope
   → decline with a reasoned reply · ambiguous → ask) — never blindly obey; fix red CI;
   rebase onto its base as the base advances — all with **verified `gh` replies** (no
   silent fixes), until the **human merges**. On merge it cleans up and retires.
5. Re-arm the **ScheduleWakeup loop (180s)** each turn; dispatch tasks as their base
   branches appear (no merge gate). End the loop only when the **plan PR is merged into
   `main`** and every task is `done`/`failed`.

Be conservative and honest: **by default never merge** (leave it to the human) — unless
the user has explicitly authorized merging for a named task/wave/cycle, in which case
honor exactly that scope (green, non-draft only) and log it; never touch main; and if a
task's gate can't go green, leave its PR as a draft with a clear blocker note.
