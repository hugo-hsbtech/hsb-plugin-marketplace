---
description: Autonomously execute a parallel cycle plan — flows end to end (no merge gates), per task drives worktree→branch→PR (stacked on its blocker, joined for multi-blocker tasks, else integration) then monitors comments/CI/conflicts until it merges. Never pushes to main; never merges the plan PR; merges a task PR only under an intact human approval. Questions it can't settle become answerable open decisions on the PR.
argument-hint: "<path to a cycle-plan .md (docs/plans/proposed/<datetime>-<slug>-<task-id>.md, or leave empty to use the plan in context / run /cadence:plan first)>"
---

Load and follow the `cadence-executor` skill (plugin `cadence`) to autonomously execute
a parallel cycle plan. This **implements and drives PRs to merge-ready and FLOWS end
to end** — it never freezes a task waiting for another's PR to merge (dependencies are
expressed by PR base: stacked on a single blocker, a **join branch** carrying all
blockers for a multi-blocker task, else the integration branch). Hard rules: **never
push or commit to `main`; never merge the plan PR (that gate is the human's); merge a
task PR only under an intact human approval or an explicit user grant; a PR is a draft
only while it is genuinely not reviewable — that call is yours, never the user's; and
keep monitoring each PR while it is open.**

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

## Dependencies (checked at the skill's preflight gate)

- **superpowers plugin — REQUIRED.** The per-task agents run `superpowers:*` skills
  end to end. If it isn't installed, the preflight STOPS the run with install
  instructions — never start without it.
- **graphifyy — optional.** When the `graphify` CLI (or `graphify-out/graph.json`)
  is present, spec agents ground their code checks in the local knowledge graph
  (cheaper + deterministic). Absent → normal file-reading analysis.

## What to do

Follow the `cadence-executor` skill end to end. **You run as a thin top orchestrator
that delegates each task to its own per-task orchestrator agent — you don't
implement, monitor, fix, or write task state yourself.** Each spawned agent reads
the skill's `references/task-agent.md` (pass its path in the brief) — that file is
the per-task playbook.
1. Locate-or-create the run **state directory**
   `.cadence/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/` (`run.json` +
   `tasks/<id>.json`; glob `*-<slug>-cycle/run.json` + matching `planPath` to resume;
   per `references/execution-state.md`). Open the integration branch + plan PR.
2. **Each tick: change detection first, then spawn only where there is work.** Run
   ONE batched read-only GraphQL call over all the cycle's open PRs **and every base
   ref**, diffed against `run.json.prSnapshot`/`refSnapshot` — an idle `open` task with
   **no delta spawns nothing**. Three things are never "no delta": a **moved base ref**
   (that's how a merge you just made reaches its dependents — their own PR fields don't
   change), **`mergeable: UNKNOWN`** (GitHub computes it lazily; re-query until it
   resolves), and a PR that is **conflicted/behind** (top priority, every tick until
   clean, and the run stays HOT). Then spawn one `Agent` per IDLE active task with work, in a single
   message — `pending`→spec (verifies-and-extends the plan brief instead of
   re-analyzing; **fuses straight into implement for `trivial`/`low` complexity**),
   `specified`→implement, `open` **with a delta**→monitor, `merged`→cleanup
   (recovery only); **skip any task whose agent is still running**
   (`specifying`/`implementing`/`fixing`) so you never tick a PR mid-round-trip
   (idle-gating). **Pick each agent's model by phase:** spec/analysis → **Opus, high
   effort** (always); implement → by the complexity the spec found (high → Opus/medium,
   medium → Sonnet; trivial/low is normally absorbed by the fused spec agent);
   monitor/cleanup → Sonnet. Don't run everything on Opus, and
   don't run analysis on a cheap model. Spec/implement/fix agents run in the background.
   Each agent
   resumes from its `tasks/<id>.json`, owns a **durable git worktree** with a
   **descriptive** `cadence/<slug>-t<id>-<task-slug>` branch (the slug reflects what the
   task does) off its **base** — the integration branch (no blockers), its **single
   blocker's branch** (stacked), or its own **join branch** (integration + all its
   blockers merged, built by the agent itself for a 2+-blocker task). It advances its task one
   step: spec (superpowers brainstorming→writing-plans, decides complexity) → implement
   (TDD, auto-approving gates) → open its PR → (later, when settled) monitor/fix →
   cleanup. It is the **sole writer of its `tasks/<id>.json`** and does all its own
   `gh` work, and it **dies when its PR
   merges**.
3. **Flow, don't gate on merges — and audit it every tick.** A task is dispatched as
   soon as the branches it depends on **exist** — a stacked child the moment its
   blocker's branch is pushed, a multi-blocker task as soon as it can build its join.
   Anything held because another PR hasn't merged is a **defect**: convert it (build or
   refresh the join, merge the base in, re-target) that same tick. Each task PR targets its base,
   never `main`. **Draft means "not reviewable yet"** (agent in flight, red gate/CI, no
   self-review, unwritten body, unanswered blocking decision) — *not* "not mergeable
   yet"; being stacked is **not** a reason to stay draft. Un-draft yourself the moment
   the readiness checklist passes, and **never ask the user whether to**. PR bodies
   follow `references/pr-template.md`, **sized to complexity** (a simple low-complexity
   change gets a short body, not the full mermaid+UAT template), didactic, referencing
   siblings by **PR number**, not bare task ids.
4. The per-task agent monitors its own PR every tick: **judge** each review/comment on
   its merits (agree → fix · better alternative → fix differently · wrong/out-of-scope
   → decline with a reasoned reply · ambiguous → ask) — never blindly obey; fix red CI;
   merge its base in as it advances (never a force-push on an open PR) and re-check that the diff still matches its scope; harvest answers to its open decisions — all
   with **verified `gh` replies** (no silent fixes). When a **human approves** its PR
   and every guard holds (approval intact and not stale, no open decision, all comments
   answered, CI green, mergeable clean), it merges its own PR into its base, posting
   the authorization first. On merge it cleans up and retires.
5. **Says where everything is, and pings you only when it matters.** Announce the run
   dir, plan doc, integration branch + plan PR, and the report path once at run open,
   then carry a one-line footer on every turn (`Run: … · report: … (/cadence:report)`).
   If a `PushNotification` tool is available, send **at most one per turn** and only on
   a transition worth interrupting for — the cycle finishing (with the report path), a
   new blocking decision, the plan PR ready for the user's merge, or a failed task.
   Never for routine progress (a PR opened, CI green, a base sync, a quiet tick).
6. **A question you can't settle becomes an open decision, not a buried default.** Post
   it on the PR as a numbered `D<n>` comment with real options, the provisional default
   and an answer protocol; pin it at the top of the PR body; track it in the task file;
   keep the PR draft while it's blocking; and repeat it in **every** turn summary's
   "needs you" list with its link. Never ship a pending question silently, and never
   turn it into a blocking prompt to the user.
7. Re-arm the **adaptive ScheduleWakeup loop** each turn — 180s while hot (agents in
   flight or changes detected), doubling per quiet tick up to `maxSeconds`
   (default 1800) while everything is parked on humans; any activity snaps it back
   to 180s. Dispatch tasks as their base branches appear (no merge gate). End the
   loop only when the **plan PR is merged into `main`** and every task is
   `done`/`failed`.

Be autonomous about process and honest about state: **decide** drafts, bases, base syncs
and readiness yourself; **merge a task PR only** under an intact human approval that
still describes the code (or an explicit user grant for a named scope), always posting
the authorization on the PR first; **never merge the plan PR** and never touch `main`.
Report each PR's real state — never "all ready for review" while any PR is a draft —
and end every turn with the "needs you" list (open decisions with their answer links,
parked reviews, failures, the plan PR when it's ready). If a task's gate can't go green,
leave its PR as a draft with a clear blocker note.
