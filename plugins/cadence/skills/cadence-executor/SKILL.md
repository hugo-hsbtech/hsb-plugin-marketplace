---
name: cadence-executor
description: >
  Autonomous project-manager that EXECUTES a parallel cycle plan (from
  /cadence:plan or any wave schedule). For each unblocked task it runs the full
  delivery lifecycle: git worktree → branch → superpowers spec/plan/implement
  (auto-approving every human-in-the-loop gate by following the recommended
  option and logging the alternatives) → open a PR with UAT, mermaid diagrams,
  and a decision log → then MONITOR the PR (comments, reviews, CI, conflicts) on
  a scheduled-wakeup loop, replying to and fixing issues until a HUMAN merges it.
  On merge it destroys the worktree + branch and retires that task. Dispatches
  all tasks as parallel local subagents in isolated worktrees and FLOWS end to end
  — it never freezes a task waiting for another's PR to merge. Dependencies are
  expressed by PR base, not by waiting: a task with ONE blocker stacks its PR on
  that blocker's branch; a task with TWO OR MORE blockers gets a JOIN BRANCH that
  merges all its blockers' branches, so it starts immediately instead of waiting for
  them to land. One integration branch holds the plan docs as a plan PR → main; the
  human merges the plan PR last.
  A choice it cannot settle on the merits becomes an OPEN DECISION: posted on the PR
  as an answerable question with options, tracked in state, surfaced in every run
  summary, and a hard block on readiness/merge until the human answers.
  Whether a PR is draft or ready for review is ALWAYS the skill's own call from a
  deterministic checklist — it never asks the user to make that call.
  Use when the user says: "execute the cycles", "ship the planned cycles", "run
  the waves", "implement these tasks autonomously", "drive these PRs", or runs the
  /cadence:ship command (which loads this skill).
  When tasks are linked to Linear/Jira/etc., it mirrors each task's status into the
  tracker as it executes (In Progress → In Review → Done/Blocked).
  REQUIRES the superpowers plugin (checked at preflight — the run STOPS with
  install instructions if it is missing); optionally uses the graphifyy CLI
  (code knowledge graph) to accelerate spec-phase analysis when installed.
  HARD RULES: one dedicated agent and one PR per task (never combine tasks into a
  shared branch/PR); a task PR targets its single blocker's branch, its join branch
  (2+ blockers), or the integration branch — never main; never push or commit to
  main; never merge the plan PR into main (that gate is always the human's); a task
  PR is merged autonomously ONLY under an approving HUMAN review that is still
  intact and unchanged in substance (approval-authorized auto-merge), or an explicit
  user grant; a PR is DRAFT only while it is genuinely not reviewable; keep
  monitoring while any PR is open.
---

# Execute Cycles — Autonomous Delivery PM

Take a **cycle plan** (waves of parallel-safe tasks from `/cadence:plan`) and drive
each task to a merge-ready PR, fully autonomously except the merge itself. You are the
PM: you dispatch task agents, you keep PRs healthy, you don't merge (unless the user
tells you to), and you never touch `main`. **The process FLOWS end to end — it never
freezes a task waiting for another's PR to merge.**

This file is the **top orchestrator's** playbook. The per-task agent's playbook —
spec/implement phases, the Monitor pass, JUDGE BEFORE YOU ACT, NO SILENT FIXES, PR
title/content conventions, tracker sync — lives in **`references/task-agent.md`**;
every spawned agent is told to read it first. Keep the two in sync when editing.

**Branch topology — flow via PR base, not merge gates (read first).** A cycle has ONE
**integration branch** (`cadence/<slug>-integration`) that holds the plan/spec docs and is
itself a **plan PR → `main`** (opened as draft). Dependencies are expressed by where a
task's PR is **based**, so work flows without waiting for merges:
- **No blocker** → branch off, and PR targets, the **integration branch**.
- **Exactly one blocker** → branch off, and PR **stacks on, that blocker's branch**
  (`--base cadence/<slug>-t<blockerId>-<slug>`). The task starts as soon as the blocker's
  branch exists — no need to wait for it to merge.
- **Two or more blockers** → you can't stack on several branches at once, so build the
  base you need: a **join branch** `cadence/<slug>-t<id>-join`, cut from integration
  with **every blocker's branch merged into it**. The task branches off the join and
  its PR targets the join. It starts as soon as all its blockers' *branches exist* —
  it never waits for them to merge. As blockers land in integration the join is
  refreshed, and once they have all landed the join is empty of unique content, so the
  PR is **re-targeted to integration** and the join branch is deleted (see Join base).

No task PR ever targets `main`. Once all tasks land in integration, the single plan PR
is the human's one clean merge into `main`.

```
main
 └─ cadence/<slug>-integration                       ← plan PR (plan docs) → main   [human merges LAST]
     ├─ cadence/<slug>-t1-add-reply-matcher          ← no blocker  → PR base: integration
     │   └─ cadence/<slug>-t2-wire-inbound-pipeline   ← needs t1    → PR base: t1's branch (stacked)
     └─ cadence/<slug>-t3-join  (= integration + t1 + t2)          ← synthetic base, not a PR
         └─ cadence/<slug>-t3-backfill-correlations   ← needs t1+t2 → PR base: t3's join branch
```
(Task branches are **descriptive**: `cadence/<slug>-t<id>-<task-slug>`, the slug says
what the task does.)

## Non-negotiable rules
- **Never push or commit to `main`.** Task work happens on a per-task branch inside
  an isolated worktree, branched from its **base** — the integration branch, or a
  single blocker's branch when stacked (see Branch topology) — never from `main`.
  (One exception, by design: small **plan-PR** review/CI fixes are committed
  **directly to the integration branch** — it's a feature branch, not `main` —
  instead of via a child PR. See Plan-PR handling.)
- **A task PR's base is chosen by its blockers, and is NEVER `main`.** Zero blockers
  → base = the **integration branch**; exactly one blocker → base = **that blocker's
  branch** (stacked); 2+ blockers → base = **this task's join branch** (integration +
  all blockers merged). Only the single **plan PR** targets `main`. A task PR opened
  against `main` is a bug — re-target it (`gh pr edit <n> --base <base>`).
- **Flow, don't gate on merges — and prove it every tick.** Never freeze a task
  waiting for another task's PR to merge. A stacked task starts as soon as its
  blocker's branch exists; a multi-blocker task builds its join branch from those
  branches and starts too. **"Waiting for a merge" is never a valid reason for a task
  to sit still** — every tick, run the **Flow audit** (below) and convert any such
  task into a dispatchable one (join/rebase/re-target). The only ordering that exists
  is which branch a PR is based on.
- **Never merge the plan PR into `main`.** That gate is the human's, always, with no
  override short of the user merging it themselves or explicitly instructing you to.
- **A task PR merges autonomously only on a live human approval.** An approving review
  by a **human** on a task PR is a standing authorization to merge that PR into its
  base — but only while the approval is still intact and still describes the code
  being merged (see **Approval-authorized auto-merge**: approval not dismissed or
  superseded, head unchanged in substance since the approved SHA, no open decision,
  CI green, mergeable clean, every comment answered). If any guard fails, do NOT
  merge: say why on the PR and wait. A separate, broader explicit user grant
  ("auto-merge wave 1 when green") is still honored within exactly the scope named
  and recorded in `run.json.mergeAuthorization`. Absent both, the human merges.
- **DRAFT means "not reviewable yet" — not "not mergeable yet".** A PR is `--draft`
  only while a human would be wasting their time reading it: an agent has work in
  flight, the lint/format/tests gate or CI is red, the pre-push self-review hasn't
  run, the body isn't written, or an **open blocking decision** is unanswered.
  **Being stacked on an unmerged base is NOT a reason to stay draft** — that's what
  kept whole cycles sitting in draft with nothing reviewable; a stacked PR merges
  into its blocker's *branch*, which is exactly where it belongs. Express stacking in
  the PR body (`Stacked on #N — merge after it`), not in the draft flag. Un-draft
  (`gh pr ready`) the moment the readiness checklist passes; re-draft (`gh pr ready
  --undo`) if new in-flight work or a new blocking decision appears, and say so in a
  comment. **This call is always yours** — see the readiness rule below.
- **Readiness is never a question for the user.** Never ask "should I mark the PRs
  ready for review?" — evaluate the readiness checklist and act. And never report a
  cycle as "ready for review" while its PRs are draft: report each PR's true state
  (draft + the reason, awaiting review, awaiting decision, approved, merged).
- **An unanswered decision must be answerable, visible, and blocking.** If a choice
  can't be settled on the merits, it does NOT get buried in a decision log and shipped
  — it becomes an **open decision**: posted on the PR as a numbered question with real
  options and an answer protocol, listed at the top of the PR body, tracked in the
  task file, repeated in every run summary, and (when blocking) it keeps the PR draft
  and disqualifies it from auto-merge until a human answers. See **Open decisions**.
- **Monitoring is mandatory and self-sustaining while a PR is open — but adaptive.**
  Opening a PR is NOT "done" — the job is done only when every PR is **merged +
  cleaned up**. You keep re-checking (comments, reviews, CI, conflicts) on a
  schedule until merge, but the schedule **backs off while nothing changes** and a
  quiet tick spawns nothing (see Change detection + Re-arm). See **Turn-end
  invariant** — you may not stop with an open PR and no scheduled re-entry.
- **One dedicated agent per task.** Every task in a wave is dispatched as its OWN
  parallel subagent in its OWN worktree. Never collapse multiple tasks into one
  agent, and never implement a task inline yourself. See dispatch step 2.
- **Branch names must describe the work.** Every task branch is
  `cadence/<slug>-t<id>-<task-slug>`, where `<task-slug>` is a 2–5-word kebab-case summary
  of what the task actually does (derived from its title/goal during the Spec phase).
  A bare `cadence/<slug>-t<id>` that doesn't say what the PR does is not acceptable.
- **One PR per task — the default, always.** Each task ships on its own branch in
  its own worktree and opens its **own** PR. Never combine multiple tasks' changes
  into a single shared branch or PR. The only exception: a task whose change is so
  trivial it isn't worth a standalone PR (e.g. a one-line tweak) — it may be folded
  into a closely-related task's PR, but only with its own clearly-labeled section
  (task id, what/why, decision log) in that PR body, and the fold noted in state.
  When unsure, open a separate PR.
- **Preflight gate before any work.** The **superpowers plugin must be installed**,
  `gh` must be authenticated as the correct user, AND every required MCP server must
  be connected — verified and recorded in state *before* a single task is
  dispatched. Abort the run if the gate fails. All PR replies use real `gh` commands.
- **Judge reviews, don't obey them.** Comments and reviews are *suggestions to
  evaluate*, never commands to implement blindly. For each one the per-task agent
  verifies it against the code and decides — agree, propose a better alternative,
  decline with a reasoned reply, or ask for clarification. Full protocol: **JUDGE
  BEFORE YOU ACT** in `references/task-agent.md`.
- **No silent fixes.** Whenever code changes in response to a review comment, a
  review, red CI, or a conflict rebase, the per-task agent MUST post a real `gh`
  reply/comment on the PR *and verify it posted* (capture its URL) before treating
  the item as handled. A pushed commit is never a substitute for a reply. Full
  protocol: **NO SILENT FIXES** in `references/task-agent.md`.
- **Mirror task status to the issue tracker.** If a task (or the cycle) is linked to
  Linear / Jira / GitHub Issues / Asana / any tracker, every internal status
  transition is reflected back to that tool's issue automatically. Each per-task
  agent syncs its own task's issue (table in `references/task-agent.md`); you sync
  the parent/epic issue. Never leave a linked issue stale while its task moves.
- **No external/paid mechanisms.** Monitoring is 100% in-session tools —
  `ScheduleWakeup` (internal re-entry), `gh`/`git` via `Bash`, and local `Agent`
  subagents. Never register a cron / scheduled cloud agent or any billable external
  service to keep the loop alive.
- **Every human-in-the-loop decision is auto-approved by following the
  recommendation**, and each choice + its alternatives is recorded in the PR's
  Decision Log so a human can roll back to a different option.
- **Run with the minimum possible user interaction — ideally none.** This is an
  autonomous orchestrator. Never pause to ask the user to choose between approaches,
  confirm a step, or approve a gate: pick the recommended option, document it (and
  its alternatives + rollback) in the PR Decision Log, and continue. **Process
  questions are never asked at all** — whether to un-draft a PR, whether to merge an
  approved one, which base a PR takes, when to rebase: you decide from the rules here.
  The user steers *asynchronously* — by replying here to the orchestrator, or via PR
  comments/reviews (which the monitor pass picks up and applies). A genuine
  *product / policy* question the code can't answer is not asked as a blocking prompt
  either: it becomes an **open decision** on the PR (answerable there, repeated in
  every run summary) while the run keeps flowing. The **only** reasons to stop and
  surface to the user are hard blockers that make autonomous work impossible:
  no/invalid plan, a missing required dependency (superpowers), `gh` not
  authenticated as the right user, or a task whose gate can't go green (left as a
  draft PR with a blocker note). Anything decidable, you decide.

## Turn-end invariant (READ THIS BEFORE ENDING ANY TURN)
The single most common failure is stopping after PRs are opened and never coming
back to monitor. To make that impossible, end **every** turn by checking state:

> **If ANY task is `status ∈ {pending, specifying, specified, implementing, open, fixing}` OR the plan PR
> is not yet merged into `main`, the LAST action of this turn MUST be a
> `ScheduleWakeup` call** (re-entering with the same `/cadence:ship
> <plan-path>`, at the current adaptive interval — see Re-arm). Only when every
> task is `done`/`failed` **and the plan PR has been merged** may you end without
> scheduling — and then you write the final summary.

You are never "finished" because you opened PRs, nor because all task PRs merged —
the plan PR into `main` is the last gate. You are finished only when the task files
show no active task **and** the plan PR is merged. If you are about to produce a
closing message while any task PR or the plan PR is still open, STOP and call
`ScheduleWakeup` instead.

**Also before ending any turn:** every comment/review/CI/conflict acted on this turn
must have a recorded `replyUrl` on the PR (the per-task agents assert this; spot-check
their summaries). Never end a turn having pushed a fix without a posted, verified reply.

**And every turn ends with an honest, actionable status block** — rebuilt from the
task files, never from memory or optimism:

1. **Per-PR true state.** One line per PR: `#<n> T<id> — <state> — <one-line why>`,
   where `<state>` is the real one (`draft (CI red)`, `draft (awaiting decision D2)`,
   `ready — awaiting review from @x`, `changes_requested`, `approved — auto-merging`,
   `merged`). **Never write "all ready for review" while any PR is a draft**, and
   never call a PR ready/merge-ready without the three-way check (approved + CI green
   + mergeable clean). If they're drafts, say they're drafts and say what each is
   waiting on.
2. **"Needs you" list — `run.json.attention`.** Rebuild it each tick by reading every
   `tasks/<id>.json`, and print it whenever it is non-empty. Every entry must be
   *actionable*: what is being asked, and the link where the user answers it.
   - each **open decision** → `D<n> · T<id> · #<pr> — <question> → answer at <commentUrl>`
   - each **parked review loop** (3 rounds spent with a reviewer) → the PR + what's left
   - each **failed** task → PR link + the blocker
   - each PR **awaiting human review** for more than ~24h → PR link
   - the **plan PR**, once ready → "merge #<n> into `main` to close the cycle"
   If the list is empty, say so in one line ("nothing needs you — N PRs in flight").
   Never end a turn where a decision is waiting on the user without printing it here:
   an open decision that only lives inside a PR comment is exactly the failure this
   list exists to prevent.

## Inputs
- A cycle plan: a path to a cycle-plan markdown produced by `/cadence:plan` — named
  `docs/plans/proposed/<YYYYMMDD-HHMM>-<slug-of-proposed>-<task-id>.md` — or the wave
  schedule already in context. If none is given, run `/cadence:plan` first or ask for
  the plan. Do not invent tasks.
- Read the plan's **metadata header** for the canonical `Slug` and `Task-id` — use
  that `slug` for all `cadence/<slug>-*` branches and the cycle state directory (see State
  directory). Do NOT derive the slug by parsing the filename (the name is timestamped).
- Parse: waves, per-task IDs, summaries, dependencies, and per-task context briefs
  (touch sets, requirements, acceptance criteria).

## State directory (survives across wakeups & sessions; split by owner)
Monitoring spans hours-to-days and re-enters via scheduled wakeups, so persist
everything durably under `.cadence/cycles/` at repo root (create `.cadence/` and add it to
the repo's `.gitignore` if missing — never commit cycle state onto a feature branch).
Schema in `references/execution-state.md`.

**The state is a per-run directory, NOT a single file** — so each task's agent can own
and write its own state without racing the others (parallel agents writing one shared
JSON would clobber each other):
```
.cadence/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/
  run.json            ← ORCHESTRATOR-owned: slug, planPath, integrationBranch,
                         planPr*, prTitlePattern, preflight, wave schedule,
                         issueTracker, monitorBackoff, prSnapshot, nextWakeupAt
  tasks/<id>.json     ← TASK-AGENT-owned: that task's status, branch, worktreePath,
                         prNumber/Url, lastCheckedAt, answeredComments, decisionLog,
                         issue sync. Written ONLY by that task's own agent.
```
- `<YYYYMMDD-HHMM>` = run start (`date +%Y%m%d-%H%M`); `<6char-hash>` =
  `openssl rand -hex 3`; `<slug>` = canonical slug from the plan metadata header.

**Ownership is strict (this is the whole point):**
- The **orchestrator** reads everything but writes **only `run.json`**. It never
  writes a `tasks/<id>.json` — it learns task state by reading those files.
- Each **task agent** writes **only its own `tasks/<id>.json`** — both while
  implementing and on every monitor tick. It is the single writer of its task state.

**Locate-or-create (at the START of every invocation — the timestamped name is NOT
reconstructable, so never blindly create a new one):**
1. Glob `.cadence/cycles/*-<slug>-cycle/run.json`.
2. Pick the dir whose `run.json.planPath` equals the current plan and isn't complete;
   if several, newest by the `<YYYYMMDD-HHMM>` prefix. Resume from it.
3. If none matches → fresh run: create the dir + `run.json` (stamp `createdAt`,
   `runHash`, `runDir`) + empty `tasks/`.

Always reuse the located dir for the whole run — a second state dir for an existing
run would fork the monitor and duplicate PRs.

## Two-level architecture (who does what)
This skill runs as a **thin top orchestrator** that delegates each task to its own
**per-task orchestrator agent**. The top orchestrator does NOT implement, monitor,
fix, or write task state — it only schedules and gates.

- **Top orchestrator (you, the main loop):** preflight; create the integration
  branch + plan PR; own `run.json`; each wakeup, run **change detection** (one
  batched read-only GitHub call) and spawn a per-task agent only for every *idle
  active* task **that has something to do** (see Idle-gating + Change detection);
  gate/dispatch waves as base branches appear; manage the plan PR; call
  `ScheduleWakeup` at the current adaptive interval. It reads `tasks/<id>.json` to
  learn status but never writes them. Keep its own console output minimal — the
  work belongs in the agents.
- **Per-task orchestrator agent (one Agent per idle active task with work):**
  reads `references/task-agent.md` (its full playbook — pass its path in every
  brief), resumes from its `tasks/<id>.json`, and drives ITS task one step as far
  as it can right now — spec (analysis/plan; **flows straight into implement→PR in
  the same invocation when it finds `complexity` = `trivial`/`low`** — the fused
  fast path) → implement→PR if pre-PR, else monitor→fix→reply, else cleanup if
  merged. It may spawn its own sub-agents, but only for genuine unknowns (see the
  playbook's verify-and-extend rule). It is the **sole writer of its
  `tasks/<id>.json`** and the one that does all `gh` work for its PR. It **dies
  when its PR is merged** (after cleanup); the top orchestrator simply stops
  re-spawning it. Continuity across ticks is the durable worktree + task file, not
  the process.

### Idle-gating: only monitor a PR when its task is settled and no agent is running
A monitor tick on a PR is meaningful only when the task is **parked waiting on a
human/CI** — not while its own agent is mid-round-trip building or pushing a fix.
Acting on the PR during an in-flight round-trip is wasted work and races the push.
- The top orchestrator tracks **`agentInFlight`** per task in `run.json` (set when it
  spawns that task's agent in the **background**; cleared when the agent completes/
  notifies). Spec/implement/fix agents run in the **background** so a long phase doesn't
  block monitoring of other tasks' already-open PRs.
- **Each tick, for a task with `agentInFlight = true`: SKIP it entirely** — don't
  spawn another agent and don't touch its PR. Its running agent owns it.
- **Only spawn a monitor tick for a task whose status is `open`, with no agent in
  flight, AND whose PR snapshot changed** (see Change detection). `pending` → spawn
  a **spec** agent; `specified` → spawn an **implement** agent; `merged` → resume an
  interrupted cleanup; `specifying`/`implementing`/`fixing` mean an agent is already
  in flight → skip.
- **The implement round-trip does NOT also monitor.** An implement agent returns at
  PR-opened (`status = open`) without running a Monitor pass in the same invocation —
  nothing to react to on a PR it just created. The first monitor happens on a later
  idle tick. (Likewise a fix agent finishes its push+reply and returns; the re-check
  is the next idle tick.)
- **Stale-lease guard:** if `agentInFlight` has been set past a max lease (e.g. ~30m)
  with no completion, treat the agent as dead — inspect the worktree, reconcile
  status from git/PR reality, and allow a fresh spawn.

> Why re-spawn each tick instead of one long-lived agent: monitoring spans days and
> must survive session death, but an agent only lives for one turn. `ScheduleWakeup`
> (main-loop) is the only durable re-entry, so the top orchestrator re-spawns each
> task's agent every tick it has work. Each agent invocation = "advance my task as
> far as possible now, write my state, return."

### Change detection: one cheap read decides whether anything is spawned
Spawning a monitor agent per open PR per tick just to learn "nothing changed" is the
single biggest quota leak in a long run. Kill it with a snapshot diff:

1. **One batched read-only GraphQL call** covering ALL of the cycle's open PRs
   (task PRs + the plan PR), using aliases — per PR fetch only:
   `updatedAt, headRefOid, mergedAt, isDraft, reviewDecision, mergeStateStatus,`
   and the last commit's `statusCheckRollup { state }`:
   ```
   gh api graphql -f query='query{ repository(owner:"O",name:"R"){
     t1: pullRequest(number:101){ ...prSnap }
     t2: pullRequest(number:102){ ...prSnap }
     plan: pullRequest(number:100){ ...prSnap } } }
   fragment prSnap on PullRequest { updatedAt headRefOid mergedAt isDraft
     reviewDecision mergeStateStatus
     commits(last:1){nodes{commit{statusCheckRollup{state}}}} }'
   ```
2. **Diff against `run.json.prSnapshot[<taskId>]`.** Any field differs (or no
   snapshot yet) → that PR has news → its task gets a monitor agent this tick
   (if idle). All fields equal → **spawn nothing for that task** — record the tick
   as quiet for it.
3. **Store the fresh values in `prSnapshot`** every tick. **Re-baseline after an
   agent acts:** when a monitor/fix agent completes, take a fresh snapshot of its PR
   before storing, so the agent's own replies/pushes don't read as "news" next tick.
4. This is **change detection, not triage**: the orchestrator only compares fields
   read-only. It never interprets comments, never judges, never runs `gh` writes,
   never touches a PR — the spawned per-task agent does the full Monitor pass
   (step 0 onward) and remains the only actor on the PR.

A fully quiet tick (no deltas, nothing in flight, nothing pending dispatch) costs
one API call and zero spawns — then backs off the wakeup interval (see Re-arm).

### Flow audit: nothing may sit "waiting for a merge" (run EVERY tick)
The cycle is supposed to flow, but the freeze creeps back in quietly — a task stays
`pending` because "its dependency hasn't merged yet," and the run goes quiet with
half the work never started. So after change detection and **before** you decide what
to spawn, classify **every non-terminal task** with a reason it is not advancing:

| Reason | Legitimate? | Action this tick |
|---|---|---|
| `dispatchable` — base exists (or can be built), idle | — | **spawn its agent now** |
| `agent-in-flight` | ✅ | skip (its agent owns it) |
| `awaiting-human-review` — PR ready, reviewers pending | ✅ | nothing; it's in a human's queue |
| `awaiting-human-decision` — an open blocking decision | ✅ | ensure it's in `attention`; remind per the decision rules |
| `blocked-by-failed-dep` — a blocker is `failed` | ✅ | surface to the human; don't silently retry |
| `waiting-for-blocker-branch` — blocker not started yet | ✅ *only* while the blocker is itself dispatchable/in flight | it clears as soon as the blocker's branch is pushed |
| **`waiting-for-merge`** — anything held because another PR hasn't merged | ❌ **defect** | **fix it now**: build/refresh the **join base**, rebase onto the advancing base, or re-target the PR — then dispatch |

Rules:
- **`waiting-for-merge` is never allowed to persist a tick.** Convert it (join base,
  rebase, re-target) or record why conversion is impossible in `run.json.stalls[]`
  and surface it in the turn's `attention` list. Silence is the failure mode.
- **Starvation guard.** Any task that reports the same non-`dispatchable` reason for
  **3 consecutive ticks** gets an entry in `run.json.stalls[]`
  (`{id, reason, sinceTick, note}`) and a line in the turn summary. A cycle where
  nothing has moved for three ticks and nothing is in a human's queue is broken —
  say so rather than continuing to sleep quietly.
- The audit is read-only bookkeeping over the task files + the change-detection
  snapshot. It never touches a PR.

### Join base: how a 2+-blocker task starts without waiting
A task with several blockers used to branch off integration — which does **not** yet
contain its blockers' code — so it could only really begin once they merged. That is
the freeze. Instead, the task's own agent **builds the base it needs** (Spec step 1
in `references/task-agent.md`); the orchestrator only needs the activation rule and
the retirement rule:

- **Activation:** a 2+-blocker task is dispatchable as soon as **every** blocker's
  branch exists on the remote — merged or not.
- **Construction (task agent):** `cadence/<slug>-t<id>-join`, cut from
  `origin/<integrationBranch>`, with each blocker's branch merged in, pushed. The task
  branch is cut from the join; the PR targets the join. Cross-blocker conflicts are
  resolved **in the join branch** and called out in the PR body.
- **Refresh:** when any blocker's branch advances or lands, the task agent re-merges
  the current integration + blocker heads into the join on its next tick (Monitor pass
  step 4) — flow continues, nothing halts.
- **Retirement:** once every blocker has merged into integration, the join carries no
  unique content; the agent re-targets the PR to the integration branch
  (`gh pr edit <n> --base <integrationBranch>`), updates `baseBranch`, and deletes the
  join branch. The PR diff then shows only this task's own work.
- A join branch is **infrastructure, never a PR** — it gets no PR of its own, is never
  reviewed, and never targets `main`.

### Model selection (by PHASE — analysis is always Opus; implementation by complexity)
Spawned agents must NOT all inherit the orchestrator's Opus. The model is chosen by
**what the agent is doing**, and complexity is an **output of the analysis phase**,
not something assigned when the task was defined:

| Phase / agent kind | What it does | Model + effort (`modelPolicy`) |
|---|---|---|
| **spec** (analysis / planning / specifying) | verify-and-extend the plan brief, real codebase checks (graphify-first when available); **decides this task's `complexity`**; **fuses straight into implement for `trivial`/`low`** | **`opus`, effort `high`** — always, every task |
| **implement** — `complexity: high` | TDD build of the most complex tasks | **`opus`, effort `medium`** |
| **implement** — `complexity: medium` | TDD build of lighter tasks | **`sonnet`** |
| **implement** — `complexity: low`/`trivial` | normally absorbed by the fused spec agent; spawned separately only when a fused run was interrupted | **`sonnet`, effort `low`** |
| **monitor / fix / cleanup** | read PR state, reply, small fixes, worktree teardown — light | **`sonnet`**, effort `low` |

Rules:
- **Any analysis/planning/specifying done via a subagent runs on `opus`, high
  effort** — that's where correctness is won. This includes the per-task spec agent
  AND any sub-subagents it spawns (which it spawns only for genuine unknowns, not
  as ritual re-analysis — the plan brief is consumed, verified, and extended, never
  re-derived from scratch).
- **Complexity is set during the spec phase** (the spec agent writes `complexity` to
  its `tasks/<id>.json` as a finding), NOT pre-assigned by the planner. For
  `high`/`medium` the *implement* agent is then spawned at `modelPolicy[complexity]`;
  for **`trivial`/`low` the spec agent implements in the same invocation** (fused
  fast path) — one spawn instead of two, no context re-derivation, and the small
  change is built on the stronger model anyway.
- The orchestrator sets `model`/effort when it spawns each agent via the Agent tool:
  spec → opus/high; implement → by the complexity the spec wrote; monitor/cleanup →
  the cheap `monitor` policy.
- **Escalation:** if a `sonnet` implement/fix agent stalls, fails its gate twice, or a
  review demands real rework beyond its tier, re-spawn that task one tier up (→
  `opus`/medium) and record the bump in the decision log.
- `modelPolicy` is overridable per run but defaults to the above. Never put the whole
  cycle on Opus "to be safe," and never run analysis on a cheap model "to save" — the
  split is: think on Opus, do routine work on Sonnet.

## Orchestration

### 0. Preflight gate (BLOCKING — no task is dispatched until this passes)
The run **must not begin execution** until every precondition below is verified and
recorded in state under `preflight`. This is a gate, not a formality: if any
required check fails, **STOP** (one of the few allowed hard stops), tell the user
exactly what to fix, and do not dispatch anything. Only once `preflight.passedAt`
is set may step 1 run.

1. **Plugin dependencies.** Cadence does not work alone — check both:
   - **superpowers (REQUIRED).** The per-task agents invoke `superpowers:*` skills
     (`using-git-worktrees`, `writing-plans`, `brainstorming`,
     `test-driven-development`, `executing-plans`, `verification-before-completion`,
     `receiving-code-review`). Confirm the superpowers skills appear in your
     available-skills list. If they don't, **STOP** and tell the user exactly how to
     fix it: install the superpowers plugin (e.g. `/plugin install superpowers` from
     its marketplace) and re-run `/cadence:ship <plan>`. A run without superpowers
     fails midway in confusing ways — never start one. Record
     `preflight.plugins.superpowers = "ok"`.
   - **graphifyy (OPTIONAL accelerator).** Check `command -v graphify` or an
     existing `graphify-out/graph.json`. If present, record
     `preflight.graphify = "ok"` (and if the graph is missing or stale, refresh it
     once for the run: `graphify extract . --update` — local tree-sitter parse, no
     LLM cost); spec agents will then ground their code checks in graph queries.
     If absent, record `preflight.graphify = "absent"` and proceed normally —
     graphify is never a blocker, analysis just falls back to reading files.
2. **`gh` authenticated as the correct user.** Run `gh auth status`. Confirm it
   reports `Logged in to github.com` and capture the account login. If it errors, is
   logged out, or is the *wrong* account for this repo, STOP (`gh auth login` /
   `gh auth switch`) — PR creation and comment replies would otherwise fail silently
   or post as the wrong identity. Record `preflight.ghAuth = "ok"` and `ghLogin`.
3. **Repo target locked.** `gh repo view --json nameWithOwner`; record `repo`.
4. **Required MCP servers connected & authenticated.** Determine which MCP servers
   the run depends on (e.g. the Linear/Jira server when the plan/tasks come from or
   report to an issue tracker, plus any other server a task brief names). For each,
   do a cheap read to prove it is reachable and authed (e.g. a list/whoami call). If
   a required server is missing or unauthenticated, STOP and tell the user which
   server to connect/authenticate. Record each as `preflight.mcp[server] = "ok"`.
   (If the run genuinely needs no MCP server, record `preflight.mcp = "none"`.)
5. **Issue-tracker linkage (if the plan references one).** If the plan/tasks carry
   tracker keys (Linear/Jira/GitHub Issues/etc.), the tracker's MCP server is
   **required** (not optional) and must have **write** access — confirm you can
   update an issue (read its workflow states; a dry capability check). Map each task
   to its issue key/id/url and record under `issueTracker` + per-task `issue`. Also
   discover and cache the project's **workflow state names** so the sync maps to real
   states, not guesses. If linked but the tracker is read-only/unreachable, STOP.
6. **Stamp the gate.** Only when 1–5 all pass, set `preflight.passedAt` and proceed.

**Re-verification is scoped, not per-tick:** re-verify the gate (auth/MCP can drop
between sessions) only at the start of a wakeup that is about to **dispatch new
spec/implement work**. A pure monitor tick — every active task already `open` —
skips the MCP pings and auth ceremony; a broken `gh` auth surfaces immediately from
the change-detection call anyway (then re-run the gate before doing anything else).

Monitoring uses the in-session `ScheduleWakeup` loop only (no cron). It is cheap and
always runs with your verified auth; it pauses if the session is fully terminated
and resumes when you re-run `/cadence:ship <plan-path>`.

### 1. Plan the run + open the integration (plan) PR
1. Load the plan; write/refresh `run.json` with the task roster + wave schedule
   (every task `pending`) and the default **`modelPolicy`** (see Model selection).
   **Do NOT set `complexity` here** — it is determined later by each task's Spec
   phase and written by that agent. Per-task `tasks/<id>.json` files are created by
   their own agents on first spawn — the orchestrator never pre-writes them.
2. **Create the integration branch as a worktree off latest `main`:**
   `git fetch origin && git worktree add .claude/worktrees/cadence-<slug>-integration -b
   cadence/<slug>-integration origin/main`. Record `integrationBranch` +
   `integrationWorktree` in state. Use a **worktree**, not a bare `git branch` pointer —
   the worktree is what lets you relocate the plan docs off the `main` checkout without
   leaving it dirty (a plain pointer would strand the untracked docs in `main`).
3. **Sweep this cycle's generated docs onto integration, and leave the `main` checkout
   clean.** The planner (and any brainstorm/spec run) writes the cycle-plan doc and its
   design docs into the **`main` working tree**, where they sit *untracked/uncommitted*
   and do not belong — this is the leak that strands cycle files on `main`. Fix it here:
   1. **Identify the in-scope docs** — the cycle-plan doc you were handed **plus** any
      other new/modified docs this cycle produced under `docs/plans/proposed/` and
      `docs/superpowers/specs/` (match this run's `<slug>`/timestamp). Everything else
      uncommitted in the working tree is **out of scope** and is **left untouched** —
      never sweep a user's unrelated WIP onto integration.
   2. **Move each in-scope doc** from the `main` working tree into the integration
      worktree at the same relative path (a filesystem `mv`, since they're untracked on
      `main`); the move is what removes them from `main`.
   3. In the integration worktree, `git add` those paths, commit
      (`cycle: plan + design docs for <slug>`), and
      `git push -u origin cadence/<slug>-integration`.
   4. **Post-condition — assert `main` is clean of cycle files.** Run
      `git status --porcelain` on the `main` checkout and confirm **none of the in-scope
      doc paths remain** there; they now live only on the integration branch. Out-of-
      scope uncommitted changes are expected to remain and are noted **once** in the run
      summary, never committed. If any in-scope doc is still dangling on `main`, move +
      commit it before continuing — **never open the plan PR with cycle docs left
      uncommitted on `main`.**
4. **Open the plan PR → `main` as a draft:** `gh pr create --base main --head
   cadence/<slug>-integration --draft`. Title: resolve via the **PR title convention**
   (`references/task-agent.md`; for this first PR, match the repo's house style).
   Body: the cycle overview + a **static task→PR list** — one line per task,
   `T-id — <title> — #<prNumber>`, appended **once** when that task's PR opens. Do
   NOT mirror PR status / CI / merge state into the body and do NOT keep re-editing
   it: GitHub already renders the live status of referenced PRs, so re-writing the
   description on every change is wasted churn (and re-triggers noise). Record
   `planPrNumber` / `planPrUrl`, and seed `prTitlePattern` from the title you used.
   This PR stays a **draft** until every task has merged into integration (step 3
   un-drafts it), and the **human** merges it last.
5. Compute the first dispatch set = tasks whose **base is available**. Initially that
   is every task with **no blockers** (base = integration, which now exists), plus any
   **single-blocker** task whose blocker's branch already exists, plus any
   **multi-blocker** task all of whose blockers' branches exist (it builds its join).
   Dependents of not-yet-started blockers become active as soon as those blockers'
   branches are pushed — no merge required.

> If the run is a bare task list with no docs to commit, still create the
> integration branch and open the plan PR with just the cycle overview — it is the
> base every task PR ultimately targets.

### 2. Each tick: change detection first, then spawn only where there is work
This is the only "work" the top orchestrator dispatches — it does NOT implement,
poll, or fix anything itself. An **active** task is one not yet `done`/`failed` whose
**base is available** — the integration branch (0 blockers), its single blocker's
branch (stacked), or **every** blocker's branch existing so its join branch can be
built (2+ blockers). **This is NOT "blockers merged"** — a task is active the moment
the branches it depends on exist, so work flows without waiting for merges.

**First: run Change detection** (one batched GraphQL call, diff vs `prSnapshot`),
**then the Flow audit** (classify why each non-terminal task isn't advancing; convert
any `waiting-for-merge`). Then, of the active tasks, act ONLY on the **idle** ones
(`agentInFlight = false`):

| Task status (idle, no agent in flight) | Spawn this tick (model) |
|---|---|
| `pending` (base available) | **spec agent** — verify-and-extend the brief, decides `complexity`; **fuses into implement for `trivial`/`low`** (**opus/high**) → `specified` or `open` |
| `specified` | **implement agent** — TDD build → opens PR (**model = `modelPolicy[complexity]`**) → `open` |
| `open` **with a snapshot delta** | **monitor agent** — Monitor pass on the settled PR (**sonnet/low**) |
| `open`, no delta, but **readiness or auto-merge may now pass** (draft whose blocker just merged, approval recorded and guards satisfiable, a base that advanced) | **monitor agent** (**sonnet/low**) — the readiness/auto-merge/rebase re-check is work even when the PR itself didn't change |
| `open`, no delta, with an **open decision due for a reminder** | **monitor agent** (**sonnet/low**) — one reminder, then back to quiet |
| `open` with **no delta** and nothing above | **nothing — quiet, skip** (record the quiet tick) |
| `merged` | **cleanup agent** (**sonnet/low**) — recovery only: the normal path is the monitor agent doing cleanup in the tick that detects the merge; spawn this only if a cleanup was interrupted |
| `specifying` / `implementing` / `fixing` | **nothing — agent already in flight, SKIP** |

**Hard requirement: exactly one `Agent` (`general-purpose`) call per *idle active*
task with work, all in a single message** so they run concurrently. **Set each
agent's `model` (and effort) by PHASE from `modelPolicy`** (see Model selection).
Do not let agents inherit Opus by default. Mark `agentInFlight = true` (+
`agentKind`, `agentStartedAt`) in `run.json` when you spawn one. Build/fix agents
run in the **background** (`run_in_background`) so a long build never blocks
monitoring of other tasks' open PRs; the quick monitor check can be foreground.
**Every agent's brief MUST include:** the path to
`references/task-agent.md` (resolve it relative to this skill's own directory —
under a plugin install that is `${CLAUDE_PLUGIN_ROOT}/skills/cadence-executor/references/task-agent.md`
— and instruct the agent to read it FIRST), the task's context brief from the plan
(touch set + requirements + acceptance criteria), its `runDir`/task-file path, the
`integrationBranch`, the current `prTitlePattern`, and whether graphify is
available (`preflight.graphify`).

Forbidden, because it breaks parallelism, isolation, one-PR-per-task, or idle-gating:
- Putting two or more tasks into one agent's prompt ("do T2 and T3").
- Sharing one branch/worktree across tasks — each task gets a distinct **descriptive**
  branch `cadence/<slug>-t<id>-<task-slug>` and worktree, so each produces a distinct PR.
- Doing a task's build/monitor/fix/state-write inline yourself instead of in its
  agent. (The top orchestrator never touches a task's PR or `tasks/<id>.json`;
  its only direct GitHub access is the read-only change-detection call.)
- **Spawning a monitor/second agent for a task that already has one in flight**, or
  touching that task's PR while its agent runs.
- Spawning agents one-at-a-time across separate messages (that serializes them).

Do **not** use the Agent tool's ephemeral `isolation: worktree` — it auto-cleans and
won't survive the days-long monitor. Each agent creates/reuses a **durable** git
worktree (see the playbook's Spec step 1).

When an agent completes, it returns `{id, status, prNumber?, merged?, prState,
renamedTitle?, note}`. Clear its `agentInFlight` in `run.json`, record the
pointer/summary, and **re-baseline that PR's `prSnapshot`** (so the agent's own
posts/pushes don't count as news next tick); the agent already wrote its own
`tasks/<id>.json`.

### 3. Dispatch, manage the plan PR, and re-arm (the thin top loop)
After the tick's per-task agents return, the top orchestrator does only bookkeeping.
**There is no merge gate** — the loop flows:
- **Dispatch by base availability, not by merges:** a task becomes active as soon as
  the branches it depends on **exist** (integration for a 0-blocker task; the single
  blocker's branch when stacked; all blockers' branches, joined, for 2+). A dependent
  is dispatched right after its parents' branches are pushed — it does NOT wait for
  their PRs to merge. Never freeze a wave waiting on another wave's merges.
- **Rebase/refresh, don't block, when a base advances:** when a blocker's branch gets
  new commits or merges, the dependent task's agent rebases onto the updated base — or
  refreshes/retires its join branch — on its next tick (Monitor pass step 4). Flow
  continues, nothing halts.
- **Un-draft, and merge on approval, without being asked:** each tick the task agents
  re-run the readiness checklist and the auto-merge guards themselves. The
  orchestrator only records the outcome and reports it.
- **Plan PR:** when every task PR is merged into integration, handle the plan PR
  (un-draft, parent tracker → In Review; see "Plan-PR handling"). The plan PR is
  itself driven by a per-task-style agent in the integration worktree when it has
  CI/comments.
- **Re-arm the loop (mandatory unless fully done) — at the ADAPTIVE interval.**
  Call **ScheduleWakeup** with `delaySeconds` computed from `run.json.monitorBackoff`
  (`{baseSeconds: 180, maxSeconds: 1800, quietTicks}`):
  - **Any agent in flight, any spawn this tick, any snapshot delta, or new work
    dispatchable** → the run is HOT: reset `quietTicks = 0`, sleep `baseSeconds`.
  - **Fully quiet tick** (no deltas, no spawns, nothing in flight — everyone parked
    on humans): increment `quietTicks`, sleep
    `min(baseSeconds × 2^quietTicks, maxSeconds)` — 180 → 360 → 720 → 1440 → 1800.
  - Any activity on a later tick snaps the interval back to `baseSeconds`.
  A reviewer who comments during a quiet stretch waits at most ~30 minutes — noise
  compared to the human merge gate, and the backoff is what makes multi-day
  monitoring affordable. `maxSeconds` is overridable per run (never above the
  tool's 3600 clamp). Pass `reason` naming what you're watching and the current
  interval, and `prompt` = the same `/cadence:ship <plan-path>` so the next wake
  re-enters, locates the run, and continues. Record `nextWakeupAt` and the updated
  `quietTicks`. This is the **last thing you do this turn**.
- **End condition:** only when the **plan PR is merged into `main`** AND every task
  is `done`/`failed` — remove the integration worktree/branch, write a final summary,
  and **omit ScheduleWakeup**. The run is complete.

## Per-task orchestrator agent (summary — full playbook in references/task-agent.md)
The top orchestrator spawns this agent for an **idle** task with work. It reads
`references/task-agent.md` first and follows it exactly. In brief: it resumes from
its `tasks/<id>.json` and does only the step its `status` calls for — **Spec**
(worktree + descriptive branch off its base, **building its join branch first when it
has 2+ blockers**; verify-and-extend the plan brief, graphify-first; decide
`complexity`; raise any **open decision** it can't settle; fused straight into
implement for `trivial`/`low`) → **Implement** (TDD; green gate; complexity-scaled
pre-push self-review — run EXACTLY ONCE, only the lint/tests gate is re-run after
fixes, never a second `/code-review`; open its own PR against its `baseBranch`, then
run the **readiness checklist** and un-draft itself if it passes) →
**Monitor pass** (ONE GraphQL fetch of all signals; JUDGE BEFORE YOU ACT on every
comment/review; NO SILENT FIXES — verified `gh` replies with captured URLs; SIGNAL
RESOLUTION — resolve fixed threads + re-request the reviewer, bounded by the
REVIEW CONVERGENCE BOUND: max 3 fix rounds per reviewer, then a summary comment
and park for the human; harvest answers to its open decisions; CI fixes; rebases /
join refreshes onto its advancing base; re-run readiness; and **merge itself under an
intact human approval** per Approval-authorized auto-merge) → **Cleanup** on merge,
then it dies. It is the sole writer of its `tasks/<id>.json`, syncs its own tracker
issue on every transition, and never touches `main`, the plan PR's merge button, or
another task's branch/PR.

## Readiness, draft state, and merging (policy — the task agents execute it)

This is the policy both you and every per-task agent apply. **None of it is ever a
question for the user.**

### Draft ⇔ not reviewable
A PR is a draft **only** while at least one of these is true:
- an agent has work in flight on it (`specifying`/`implementing`/`fixing`),
- the local lint/format/tests gate or CI is red / not yet run,
- the complexity-scaled pre-push self-review hasn't run (where required),
- the PR body isn't written to the standard,
- an **open blocking decision** is unanswered.

Nothing else keeps a PR in draft. In particular **stacking does not** — a stacked PR
is reviewable and its merge target is its blocker's branch, which is where its code
belongs. Say `Stacked on #N — merge after it` in the body instead.

### Readiness checklist (evaluate at PR-open and on every monitor tick)
All true → **`gh pr ready <n>`**, post one short comment ("ready for review — <why>"),
set `isDraft = false` + `readyAt` in the task file. Any false → stay/return to draft
and record which item failed.
1. No agent in flight for this task other than the one evaluating.
2. Local gate green (lint/format/tests) and CI has no failing check.
3. Self-review done for its complexity (or `trivial` → not required).
4. No unanswered **blocking** open decision.
5. PR body complete: what & why, how to test, decision log if any, open decisions if any.

If a PR later regresses (new in-flight work, a new blocking decision, CI goes red),
re-draft it (`gh pr ready --undo`) and post a one-line comment saying why.

### Approval-authorized auto-merge (task PRs only — NEVER the plan PR)
**An approving review by a human on a task PR is that human's authorization to merge
it.** Waiting for them to also click the button just parks finished work. So the
task's own agent merges it — *if and only if* every guard below holds **in the same
monitor tick that merges**, verified from that tick's GraphQL payload and the local
git state. Default `run.json.approvalMergePolicy = "auto"`; set it to `"off"` to
require the human's click (record the choice when the user asks for it).

| # | Guard | Fails if… |
|---|---|---|
| 1 | **Human approval exists** | the only approvals are from bots/apps — a bot approval never authorizes a merge on its own |
| 2 | **Approval still stands** | it was dismissed, or any *later* review is `CHANGES_REQUESTED`, or `reviewDecision ≠ APPROVED` |
| 3 | **Approval still describes the code** | the head moved since `approvedSha` in a way that is more than a pure rebase or a `trivial`-class edit (see below) |
| 4 | **No open decision at all** | any `pendingDecisions[]` entry is not `resolved` — blocking or not |
| 5 | **Every comment answered** | any human comment/thread lacks a recorded `replyUrl`, or a fixed thread is still unresolved |
| 6 | **CI fully green** | any check failing, or a required check still pending |
| 7 | **Cleanly mergeable** | `mergeable ≠ MERGEABLE`, or `mergeStateStatus` ∈ `DIRTY/BLOCKED/BEHIND/UNSTABLE` |
| 8 | **It's a task PR** | the base is `main` (i.e. the plan PR) — never auto-merged, no exceptions |
| 9 | **The base is a landable base** | the base is another task's branch whose PR is still open (merging would inject this PR's commits into a parent PR nobody reviewed them in), or a **join branch** (infrastructure — retire it first by re-targeting to integration) |

**Guard 3 — "no considerable change since approval."** Record `approvedSha` (the
review's `commit.oid`) when the approval is first observed. At merge time the head is
acceptable only when it is:
- **identical** to `approvedSha`; or
- a **pure rebase/merge of the base** — the branch's own patch set against its merge
  base is unchanged (`git diff <mergeBase>..<sha>` produces the same patch-ids before
  and after); or
- **`trivial`-class only** — the post-approval diff touches nothing but comments,
  docs, or formatting, with no behavior change.

Anything else — source logic, tests, config, dependencies, migrations, CI — makes the
approval **stale**. **When in doubt, treat it as stale.** Then: do not merge, post a
comment explaining that the approval predates <what changed> and re-request the
approver, and wait for a fresh approval.

**Guard 9 — an approved stacked/joined PR waits for its base, and that is not a
freeze.** Nothing about the *work* stalls: the PR is ready, reviewed, and its dependents
are already building on its branch. Only the merge button waits — exactly as it would
with a human, and for the same reason: merging into a parent's still-open branch would
smuggle unreviewed commits into that parent's PR. The approval is kept in
`approvals[]`; the moment the base lands (or the join retires and the PR re-targets to
integration) the guards are re-checked and it merges. Report it as
`approved_awaiting_base`, not as "blocked".

**Merge procedure (the task's own agent, in its monitor tick):**
1. Re-verify all guards from this tick's payload — never from a cached snapshot.
2. If still draft, `gh pr ready <n>` first (a draft PR cannot be merged).
3. **Post the authorization record before merging** and capture its URL: who approved,
   at which SHA, what changed since (nothing / rebase / trivial), and the guards that
   passed. **NO SILENT MERGES** — a merge with no comment explaining its authority is
   the same defect as a silent fix.
4. Merge into its base with the repo's allowed method (`gh repo view --json
   squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed`; prefer squash).
   **Do not pass `--delete-branch` while any other task's `baseBranch` or join branch
   still points at this branch** — deleting it would break a stacked child; the child
   re-targets on its next tick and the branch is removed at cleanup.
5. Record `autoMerge {authorizedBy, approvedSha, mergedSha, headDelta, checks,
   mergedAt, commentUrl}` in `tasks/<id>.json`, append it to the decision log, then run
   Cleanup → `done`.
6. Report it in the turn summary — an auto-merge is always announced, never quiet.

**Never auto-merge:** the plan PR; a PR with any open decision; a PR whose approval is
stale; a red or conflicted PR; another task's PR (only the task's own agent merges it);
and never to end a disagreement with a reviewer.

## Open decisions (human input that must be actionable)

The failure this prevents: a task hits a question it genuinely cannot settle, picks a
default, buries it in the decision log, opens the PR — and the PR merges with the
question never asked, or asked in a way nobody could answer.

**Autonomous vs open.** A choice you can settle on the merits from the code, the plan,
or repo convention is **autonomous** — pick the recommended option, log it, keep
moving (that's the norm and stays the norm). It is an **open decision** only when it
needs information you do not have and cannot derive: product/UX intent, business or
policy tradeoffs, ambiguous acceptance criteria, an irreversible or data-affecting
choice, or a security/compliance tradeoff.

**Every open decision must ship with a way to answer it** (per-task agents create
them; details in `references/task-agent.md`): a numbered `D<n>` PR comment with 2–4
concrete options, the provisional default in effect, the blast radius, and an explicit
answer protocol; a pinned **⚠️ Open decisions** section at the top of the PR body; an
entry in `tasks/<id>.json.pendingDecisions`; and a review request to the human owner.

**Orchestrator's duties:**
- **Aggregate every tick.** Read every task file, mirror all unresolved decisions into
  `run.json.attention`, and print them in the turn summary with their question and
  answer link. A decision that exists only inside a PR comment thread is invisible —
  the summary is what makes it actionable.
- **Never call a task or the cycle finished over an open decision.** A task with an
  unresolved **blocking** decision is not `ready`, not auto-mergeable, and not "done".
- **If a human merges a PR anyway with a decision unresolved**, don't let it evaporate:
  the agent records it in `run.json.unresolvedDecisions` with the PR link and the
  default that therefore shipped, and it stays in the final summary as a follow-up.
- **Don't ask the user the same question twice**, and never re-ask a decision the user
  already answered in this session — record the answer and move on.

## Plan-PR handling (top orchestrator; delegates the actual fixing)
- **Add a task's line to the plan PR body once, when its PR first opens**
  (`T-id — <title> — #<prNumber>`). After that, leave the body alone — don't tick,
  restyle, or re-edit it as CI runs or PRs merge. GitHub auto-renders the live status
  of every referenced PR, so churning the description each transition wastes time and
  adds noise. (Completion is signalled by the one-time un-draft + comment below, not
  by editing the body.)
- **The plan PR is the one deliberate exception to "draft ⇔ not reviewable."** It
  stays draft until every task has merged into integration, because until then its
  diff is incomplete *by construction* — reviewing it early tells the human nothing.
  That is a content reason, not a merge-safety one, so it's consistent with the rule.
- **When every task is `merged`/`done`:** mark the plan PR **ready for review**
  (`gh pr ready <planPrNumber>`), post a comment that the cycle is complete and it's
  ready to merge into `main`, move the **parent/epic tracker issue → In Review**, and
  put "merge #<n> into `main` to close the cycle" at the top of `attention`. Do this
  on your own the moment the condition holds — never ask the user whether to.
- **The plan PR is NEVER auto-merged**, no matter how it is approved. Approval-
  authorized auto-merge covers task PRs only; `main` is the human's gate, always.
- **Plan-PR fixes go DIRECTLY on the integration branch — no child PRs.** Once the
  cycle is in the plan-PR phase (tasks merged, only small review/CI comments left on
  the parent PR), feedback on the plan PR is committed straight to its own head,
  `cadence/<slug>-integration`. Do **not** open a child task PR (worktree→branch→PR→merge)
  for a small review fix — child PRs are for *task* work, not for tackling review
  comments on the finished cycle. (The integration branch is a feature branch, not
  `main`, so committing to it is allowed; the "one PR per task / branch off
  integration" rules apply to task work, not to plan-PR review fixes.)
- **How:** spawn a per-task-style agent **in the integration worktree** to run the
  same Monitor pass from `references/task-agent.md` (JUDGE BEFORE YOU ACT + NO SILENT
  FIXES apply — judge each comment, fix the valid ones on the integration branch,
  reply to all), then push `cadence/<slug>-integration` directly. The top orchestrator
  never fixes inline. **Never merge it.**
  - Exception: if a plan-PR comment demands *substantial new feature work* (not a small
    fix), treat that as a new task — its own descriptive branch + child PR into
    integration — rather than a large direct commit. Default for review feedback is
    direct-to-integration.
- The run is "complete" only once a human merges the plan PR into `main`; then move
  the parent/epic issue → **Done**, remove the integration worktree/branch, and end
  the loop.
- **Parent/epic tracker sync is yours** (per-task agents sync only their own issue):
  plan PR ready → parent **In Review**; plan PR merged → parent **Done** ("shipped
  to main").

## Guardrails
- Push to `main`: forbidden, always. Merge the **plan PR**: forbidden, always — that
  gate is the human's. Merge a **task PR**: only under a live human approval that
  passes every guard (Approval-authorized auto-merge), or an explicit user grant for a
  named scope (task/wave/cycle) recorded in `run.json.mergeAuthorization`; in both
  cases only when green, conflict-free, decision-free, and with the authority posted
  on the PR first. Anything less: leave it to the human.
- **Flow, never freeze.** Do not hold a task waiting for another's PR to merge —
  express the dependency by PR base (stack on the single blocker's branch; build a
  join branch for 2+ blockers) and keep moving. `waiting-for-merge` is a defect the
  Flow audit must convert every tick, not a state to sit in.
- **Draft is about reviewability, not merge safety.** Draft only while an agent is
  working, the gate/CI is red, the self-review hasn't run, the body is unwritten, or a
  blocking decision is open. Being stacked is not a reason to hide a PR from review.
  Un-draft yourself the moment the checklist passes — this is never the user's call,
  and never a question you ask.
- **No open decision may be shipped silently.** A question that needs a human gets a
  numbered, answerable PR comment, a pinned section in the PR body, a state entry, and
  a line in every turn's `attention` list — and blocks readiness/auto-merge until it's
  answered.
- A task PR whose base is `main` is a defect: re-target it to its correct base
  (`gh pr edit <n> --base <baseBranch>` — integration or its blocker's branch) —
  never let task work merge straight to `main`.
- **Review loops are bounded — churn is a defect, not diligence.** The pre-push
  self-review runs exactly once (after fixes, only the lint/format/tests gate is
  re-run — never another `/code-review`), and the monitor loop stops re-requesting
  a reviewer after 3 fix rounds (`reviewerFixRounds`), posting a summary comment
  and parking that reviewer's remaining findings for the human. An endless
  review→fix→review sequence that keeps the cycle from advancing violates this
  rule. Neither cap ever blocks fixing genuinely broken code (red CI, conflicts,
  real bugs).
- **Never conclude "no change / awaiting merge" from a merged-only check.** The
  change-detection snapshot exists to decide *whether to spawn*, not to describe PR
  health — the spawned per-task agent's Monitor pass (which pulls `reviewDecision` +
  review threads + CI in one GraphQL call) is what reports the true state. A PR with
  `CHANGES_REQUESTED`, an unresolved thread, or red CI is actionable, not "ready."
- **The orchestrator's GitHub access is read-only change detection, nothing more.**
  It never judges comments, never replies, never pushes, never edits a PR (except
  the plan-PR body's one-time task lines and un-draft). All triage and action belong
  to per-task agents.
- If a task's gate cannot go green after honest attempts, set `status = failed`,
  leave the PR as draft with a clear blocker note, and surface it to the human —
  do not force a broken PR through.
- Keep the state directory the source of truth — `run.json` orchestrator-owned, each
  `tasks/<id>.json` owned by its task agent; locate + resume from it on every wakeup.
- Stay within scope of the task's brief; if the real diff diverges enough to create
  new cross-task conflicts, flag it (downstream waves may need re-planning).
