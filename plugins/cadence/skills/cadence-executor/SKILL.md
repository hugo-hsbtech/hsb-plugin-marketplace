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
  with **every blocker's branch merged into it**. Branch your work **off the join** so
  you compile and test against your blockers' code — but the **PR targets the
  integration branch, NEVER the join**. The task starts as soon as all its blockers'
  *branches exist*; it never waits for them to merge.

  > **NOTHING EVER MERGES INTO A JOIN BRANCH.** A join is scaffolding: it merges
  > nowhere, so anything merged *into* it is stranded off integration — the source
  > branch is auto-deleted on merge, the work vanishes from the cycle, the exit gate
  > runs without it, and someone has to invent a rescue PR to carry it across. A task PR
  > whose base is a join is a **defect**: re-target it to integration immediately
  > (`gh pr edit <n> --base <integrationBranch>`), record it, surface it. This is not a
  > style preference — it is the difference between work landing and work disappearing.

  Because the PR targets integration while the branch sits on top of unmerged blockers,
  its diff **temporarily shows those blockers' files too**. That is expected, not
  pollution — declare it (see the scope check) and it shrinks to this task's own work by
  itself as each blocker merges into integration.

No task PR ever targets `main`. Once all tasks land in integration, the single plan PR
is the human's one clean merge into `main`.

```
main
 └─ cadence/<slug>-integration                       ← plan PR (plan docs) → main   [human merges LAST]
     ├─ cadence/<slug>-t1-add-reply-matcher          ← no blocker  → PR base: integration
     │   └─ cadence/<slug>-t2-wire-inbound-pipeline   ← needs t1    → PR base: t1's branch (stacked)
     └─ cadence/<slug>-t3-join  (= integration + t1 + t2)          ← synthetic base, not a PR
         └─ cadence/<slug>-t3-backfill-correlations   ← needs t1+t2 → branched off the join,
                                                     but PR base: integration
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
  task into a dispatchable one (join / base-sync / re-target). The only ordering that exists
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
- **Every PR says which task it is, everywhere it appears.** Identity header as the
  first line of every PR body (both sizes), a task→PR map table in the plan PR, and — in
  anything you write — never a bare
  task id without its PR number and a plain description, never a PR number without what
  it does. Making the human ask "which PR is T3?" is a defect. See **Task ↔ PR
  identity**.
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
- **One PR per task — always, with no exceptions.** Each task ships on its own branch
  in its own worktree and opens its **own** PR. Never combine two tasks' changes into a
  shared branch or PR, and never fold a small task into a sibling's PR — a task agent
  may not touch another task's branch or PR at all, so that "fold" was never actually
  possible.
  **Batching small work is a PLAN-time decision, not a dispatch-time one.** The planner
  emits a **prep bundle** — one task carrying several same-class scopes (`docs` /
  `config` / `schema`, never mixed), capped at ~5 scopes, with a union touch set. To the
  executor a bundle is simply *one task*: one branch, one PR, one agent. Handle it like
  any other task, with three adjustments:
  - its `complexity` is that of its **highest** scope (a `schema` bundle is never
    `trivial`, whatever the file count suggests);
  - a **`schema` bundle never skips the pre-push self-review** — minimum
    `/code-review low`, even if each migration looks small;
  - its PR body carries **one clearly-labeled section per scope** (`### Scope 2 of 4 —
    <what> (R-ids)`) with that scope's what/why, how-to-test, and roll-back note, plus
    one line stating the scopes **revert as a set**. The scope check runs against the
    **union** touch set.
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
  review, red CI, or a base sync/conflict, the per-task agent MUST post a real `gh`
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
  approved one, which base a PR takes, when to sync it: you decide from the rules here.
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

1. **Per-PR true state, in a table that says which PR is which task.** Never a bare
   `T2` and never a bare `#1207` — a reader must never have to ask which PR belongs to
   which task (see **Task ↔ PR identity**). Render:

   | PR | Task | What it does | State |
   |---|---|---|---|
   | #1207 | T2 | Wire the matcher into the inbound pipeline | draft — CI red |
   | #1206 | T1 | Add the reply-correlation matcher | ready — awaiting review from @x |

   `<state>` is the real one (`draft — CI red`, `draft — awaiting decision D2`,
   `ready — awaiting review from @x`, `changes_requested`, `approved — auto-merging`,
   `approved — awaiting base #1206`, `merged`). **Never write "all ready for review"
   while any PR is a draft**, and never call a PR ready/merge-ready without the
   three-way check (approved + CI green + mergeable clean). If they're drafts, say
   they're drafts and say what each is waiting on.
2. **"Needs you" list — `run.json.attention`.** Rebuild it each tick by reading every
   `tasks/<id>.json`, and print it whenever it is non-empty. Every entry must be
   *actionable*: what is being asked, and the link where the user answers it. Every
   entry names its PR **and what that PR does** — never a bare task id.
   - each **open decision** → `D<n> · #<pr> (<what the PR does>) — <question> → answer at <commentUrl>`
   - each **parked review loop** (3 rounds spent with a reviewer) → `#<pr> (<what it does>)` + what's left
   - each **failed** task → `#<pr> (<what it does>)` + the blocker
   - each PR **awaiting human review** for more than ~24h → `#<pr> (<what it does>)`
   - the **plan PR**, once ready → "merge #<n> into `main` to close the cycle"
   If the list is empty, say so in one line ("nothing needs you — N PRs in flight").
   Never end a turn where a decision is waiting on the user without printing it here:
   an open decision that only lives inside a PR comment is exactly the failure this
   list exists to prevent.
3. **A "where things live" footer — one line, every turn.** The user must never have to
   hunt for the run's files or ask where the report is:
   ```
   Run: <absolute runDir> · report: <runDir>/report.md (`/cadence:report` to refresh) ·
   plan: <planPath> · integration: <integrationBranch> (plan PR #<n>)
   ```
   Print the report line whether or not the file exists yet — say `not written yet
   (/cadence:report renders it any time)` when it doesn't, so the user knows it's
   available on demand rather than only at the end.
4. **Notify — but only when it's worth interrupting for.** This loop runs for days; the
   user has almost certainly walked away. If a `PushNotification` tool is available,
   send **at most one per turn**, and only for:
   - **the cycle finishing** — "cycle <slug> complete: N PRs merged · report at
     <runDir>/report.md";
   - **something new that blocks progress and only the user can clear** — a blocking
     open decision just raised, the plan PR now ready for their merge, or a task marked
     `failed`. Notify on the **transition** (the item entering `attention`), never again
     on later ticks while it sits there.

   **Never notify** for routine progress — a PR opened, CI going green, a review
   answered, a base sync, a quiet tick. A notification the user didn't need costs more
   than the one it was bundled with saved. When there's nothing new in `attention` and
   the cycle isn't done, send nothing.

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

**4. Pin the policy — an upgrade must never change the rules under a live run.**
Monitoring spans days, and `/cadence:ship` re-enters by re-reading *these files*. If the
plugin is upgraded mid-run, the re-entering orchestrator would silently adopt the new
behaviour — so a run whose human was told "merges are human-only" could start merging.
That is a promise broken by a version bump, and it is not allowed.
- **On a fresh run:** write `policyVersion` (the installed version) and an explicit
  `approvalMergePolicy` into `run.json`, and state the merge policy in the run-open
  announcement so it's auditable later.
- **On resume with no `approvalMergePolicy`:** the run predates the feature → set it to
  **`"off"` (human-only)**, log `policy.pinned`, and say so once. Never infer the new
  default into an old run.
- **On resume where the installed version ≠ `policyVersion`:** note the skew **once** —
  "run opened under 3.2.1, plugin now 3.7.0 — merge policy stays as it started; say
  'adopt the new policy' to change it" — and keep behaving as the run started.
- **Only the user changes it mid-run**, explicitly; then update both fields and record
  the authorization.

Pinned: **merge policy, draft/readiness semantics, `main` safety, and the BRANCH
TOPOLOGY (how a task's PR base is derived from its blockers)** — the promises a human
acts on. Everything mechanical (better conflict detection, cheaper ticks, richer
reports) applies immediately, because it changes nothing the user was told.

**Topology is pinned because an upgrade silently re-based a live cycle.** A run opened
under a version whose rule was "0 or 2+ blockers → integration" was resumed after the
plugin upgraded to a join-branch model, and the new model was adopted mid-flight —
against the plan doc's own topology section *and* against an explicit user instruction
already recorded in state. Four tasks' work merged somewhere the cycle could not see it
(`pagana-catalog-apps#38/#39/#40/#42`), and unwinding it took three extra PRs. So:

- **At run open, write `topology` into `run.json`** — the base rule this run will use
  for 0 / 1 / 2+ blockers, plus `source: "skill" | "plan"`. It is pinned exactly like
  the merge policy: a resumed run keeps the topology it opened with, whatever version
  is installed now.
- **The plan doc wins, and a disagreement is surfaced, never resolved silently.** If
  the plan doc states a topology (a "branching"/"topology"/§-numbered base rule) that
  differs from this skill's, take the **plan's** — the human read and approved that
  document — record `topology.source = "plan"` with the conflicting skill rule
  alongside it, log a `policy.pinned` event, and say so in one line at run open. Only
  the user resolves it the other way.
- **On resume, re-assert before dispatching anything.** If the topology you are about
  to apply differs from `run.json.topology`, that is the upgrade drifting under a live
  run: **keep the pinned one**, log an `incident` (`kind: "topology.skew"`), and put it
  in `attention`. Never let a version bump re-base a cycle that is already in flight.

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
  the same invocation only when it finds `complexity` = `trivial`** — the fused
  fast path; `low` and up stop at `specified` so the implement tier applies) → implement→PR if pre-PR, else monitor→fix→reply, else cleanup if
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
   (task PRs + the plan PR) **and every base ref** — per PR:
   `updatedAt, headRefOid, baseRefName, mergedAt, isDraft, reviewDecision, mergeable,
   mergeStateStatus,` the last commit's `statusCheckRollup { state }`, **plus the head
   OID of `main`, of the integration branch, and of every task/join branch**:
   ```
   gh api graphql -f query='query{ repository(owner:"O",name:"R"){
     t1: pullRequest(number:101){ ...prSnap }
     t2: pullRequest(number:102){ ...prSnap }
     plan: pullRequest(number:100){ ...prSnap }
     mainRef:  ref(qualifiedName:"refs/heads/main"){ target{oid} }
     integRef: ref(qualifiedName:"refs/heads/cadence/<slug>-integration"){ target{oid} }
     t1Ref:    ref(qualifiedName:"refs/heads/cadence/<slug>-t1-…"){ target{oid} } } }
   fragment prSnap on PullRequest { updatedAt headRefOid baseRefName mergedAt isDraft
     reviewDecision mergeable mergeStateStatus
     commits(last:1){nodes{commit{statusCheckRollup{state}}}} }'
   ```
   **Why the refs matter:** when the human merges a PR, the *dependents'* PRs do not
   change — their `updatedAt` doesn't move — but their **base ref does**. Without
   watching refs, a merge silently leaves every dependent behind or conflicted and
   nothing wakes up to fix it. A moved base ref is a delta for **every** PR based on it.

   **`main` is watched too, and a moved `main` is a delta for the whole cycle.** The
   repo does not stop while a cycle runs: somebody merges unrelated work into `main`,
   and the cycle is now built on a foundation that changed underneath it. In production
   an external PR renamed the tables a cycle's *already-merged* migration indexed and
   swapped the DB layout the whole chain assumed — the cycle's own gates never saw it
   (the affected suite only runs on a manual input), and it surfaced late as the plan
   PR going `CONFLICTING`. So when `mainRef` moves: spawn an agent in the **integration
   worktree** to merge `main` into integration promptly (merge, never rebase — the plan
   PR is open), scope-check the result against both parents, and treat any breakage it
   reveals as a first-class cycle finding — it goes in `attention`, gets an `incident`
   event, and lands in the report. Do **not** wait for the plan PR to go red to notice.
2. **Diff against `run.json.prSnapshot[<taskId>]`.** Any field differs (or no
   snapshot yet) → that PR has news → its task gets a monitor agent this tick
   (if idle). All fields equal → **spawn nothing for that task** — record the tick
   as quiet for it. Three exceptions that override "no delta":
   - **`mergeable: UNKNOWN` is NOT "no news."** GitHub computes mergeability *lazily*:
     right after a base moves it returns `UNKNOWN`, and a naive `UNKNOWN == UNKNOWN`
     comparison reads as quiet — which is exactly how a conflicted PR sits unnoticed
     for hours. Treat `UNKNOWN` as **unresolved**: never count that PR's tick as quiet,
     re-query next tick at `baseSeconds` (the query itself is what makes GitHub compute
     it), and if it is still `UNKNOWN` after ~3 tries, spawn the monitor anyway to
     determine mergeability locally (`git merge --no-commit --no-ff` dry run).
   - **A base ref that moved** → delta for every dependent, even if their own fields
     are untouched (see step 2b).
   - **`mergeStateStatus` ∈ `DIRTY`/`BEHIND`, or `mergeable: CONFLICTING`** → this PR
     is **top priority** and gets a monitor agent **every tick until it's clean**,
     whatever else is happening.
2b. **A merge fans out immediately — in the SAME tick.** When the snapshot shows a task
   PR merged, or the integration/blocker ref moved, spawn a sync monitor for **every
   open dependent** at once (one message, parallel agents), not one per subsequent tick.
   The human merges several PRs in a row; the dependents must all be brought current
   right away, not over the next half hour. Any of this **resets `monitorBackoff` to
   `baseSeconds`** — a moved base or a conflict means the run is HOT, never quiet.
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
| **`conflicted` / `behind`** — base moved under it (`DIRTY`/`BEHIND`/`CONFLICTING`, or `mergeable: UNKNOWN` unresolved) | ❌ **defect if it persists** | **highest priority: spawn its agent NOW**, every tick until clean. A conflicted PR is unreviewable *and* shows the wrong diff — it must never sit waiting for a human to notice |
| `awaiting-human-decision` — an open blocking decision | ✅ | ensure it's in `attention`; remind per the decision rules |
| `blocked-by-failed-dep` — a blocker is `failed` | ✅ | surface to the human; don't silently retry |
| `waiting-for-blocker-branch` — blocker not started yet | ✅ *only* while the blocker is itself dispatchable/in flight | it clears as soon as the blocker's branch is pushed |
| **`waiting-for-merge`** — anything held because another PR hasn't merged | ❌ **defect** | **fix it now**: build/refresh the **join base**, merge the advancing base in, or re-target the PR — then dispatch |

Rules:
- **`waiting-for-merge` is never allowed to persist a tick.** Convert it (join base,
  base-sync, re-target) or record why conversion is impossible in `run.json.stalls[]`
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
  branch is cut from the join — and **the PR targets the INTEGRATION branch, never the
  join** (`gh pr create --base <integrationBranch>`). Branching off the join is how you
  compile against your blockers; targeting integration is how the work reaches the
  cycle. Cross-blocker conflicts are resolved **in the join branch** and called out in
  the PR body. **A join is per task, never shared** — two tasks pointing at one join is
  how a join stops being scaffolding and becomes a base people merge into.

  > **Why this bullet is worded so insistently.** In production, a run whose plan said
  > "target integration" adopted a mid-run upgrade's join model and let four task PRs
  > (`pagana-catalog-apps#38`, `#39`, `#40`, `#42`) target — and then merge into — their
  > joins. A join has no PR, so nothing carried that work onto integration: it took a
  > four-step, three-PR recovery (`#41`, `#45`, `#46`) to unwind. **Branch off the join,
  > target integration.** If you have just built a join, re-read this line before
  > `gh pr create`.
- **Refresh:** when any blocker's branch advances or lands, the task agent re-merges
  the current integration + blocker heads into the join on its next tick (Monitor pass
  step 4) — flow continues, nothing halts.
- **Retirement:** once every blocker has merged into integration, the join carries no
  unique content — just delete it. There is no PR to re-target: the PR targeted
  integration all along, and its diff shrinks to this task's own work by itself as each
  blocker lands.
- A join branch is **infrastructure**: no PR of its own, never reviewed, **never a merge
  target**, never shared between tasks, never targets `main`.
- **Repair rule (for runs that got this wrong).** A task PR based on a join → re-target
  it to integration now. Work already **merged into** a join is stranded off integration
  — a cycle defect: surface it in `attention` and the report, and recover it by
  re-targeting the surviving branch, or (if the source branches were auto-deleted) with
  ONE clearly-labelled recovery PR from the join to integration, identified as
  `cycle-repair` with what went wrong. **Never open plumbing PRs as routine** — a PR with
  no task behind it means the topology broke, and it should read as the incident it is.

### Model selection (by PHASE — analysis is always Opus; implementation by complexity)
Spawned agents must NOT all inherit the orchestrator's Opus. The model is chosen by
**what the agent is doing**, and complexity is an **output of the analysis phase**,
not something assigned when the task was defined:

| Phase / agent kind | What it does | Model + effort (`modelPolicy`) |
|---|---|---|
| **spec** (analysis / planning / specifying) | verify-and-extend the plan brief, real codebase checks (graphify-first when available); **decides this task's `complexity`**; **fuses straight into implement for `trivial` ONLY** | **`opus`, effort `high`** — always, every task |
| **implement** — `complexity: high` | TDD build of the most complex tasks | **`opus`, effort `medium`** |
| **implement** — `complexity: medium` | TDD build of lighter tasks | **`sonnet`** |
| **implement** — `complexity: low` | TDD build of a small, well-understood change — **always its own agent**, never fused | **`sonnet`, effort `low`** |
| **implement** — `complexity: trivial` | absorbed by the fused spec agent; spawned separately only when a fused run was interrupted | **`sonnet`, effort `low`** |
| **monitor / fix / cleanup** | read PR state, reply, small fixes, worktree teardown — light | **`sonnet`**, effort `low` |

Rules:
- **Any analysis/planning/specifying done via a subagent runs on `opus`, high
  effort** — that's where correctness is won. This includes the per-task spec agent
  AND any sub-subagents it spawns (which it spawns only for genuine unknowns, not
  as ritual re-analysis — the plan brief is consumed, verified, and extended, never
  re-derived from scratch).
- **Complexity is set during the spec phase** (the spec agent writes `complexity` to
  its `tasks/<id>.json` as a finding), NOT pre-assigned by the planner. For
  `high`/`medium`/**`low`** the *implement* agent is then spawned separately at
  `modelPolicy[complexity]`; **only `trivial` fuses** — the spec agent finishes the
  change in the same invocation, because re-spawning to fix a typo costs more than it
  saves.
- **`low` does NOT fuse — and that is deliberate.** Fusing means one invocation on one
  model, and the spec model is always Opus/high, so a fused task is *implemented* on
  Opus no matter what its complexity says. Extending that to `low` inverted the whole
  cost curve: `high` built on opus/medium, `medium` on sonnet, and the *cheapest* tasks
  on the most expensive model — which is how a cycle ends up running almost entirely on
  Opus while the policy table claims otherwise. A `low` task therefore ends at
  `specified` and gets its own **sonnet/low** implement agent. The extra spawn is the
  price of the tier actually applying; `trivial` keeps the fast path because there the
  spawn genuinely costs more than the model does.
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
   Body: the cycle overview + **the cycle plan diagram** + the **cycle map table**
   (`| Task | What it does | PR | Base |`), one row appended **once** when that task's
   PR opens — the canonical answer to "which PR is which task" (see **Task ↔ PR
   identity**).

   > **Every integration/plan PR carries the cycle plan as a Mermaid diagram.** The plan
   > PR is where a human goes to understand the *shape* of the cycle, and a table of
   > rows doesn't show dependencies. Render two small diagrams from the plan doc, in the
   > body, above the map table:
   > 1. **Waves + dependencies** — what runs in parallel and what waits on what:
   >    ```mermaid
   >    graph LR
   >      subgraph W1[Wave 1]
   >        T1["T1 · add reply matcher"]
   >        T4["T4 · metrics dashboard"]
   >      end
   >      subgraph W2[Wave 2]
   >        T2["T2 · wire inbound pipeline"]
   >      end
   >      T1 --> T2
   >    ```
   > 2. **Branch topology** — where each task's PR is based, so the stacking is visible:
   >    ```mermaid
   >    graph RL
   >      main([main]); integ["integration"]
   >      integ -. "this plan PR (merged LAST)" .-> main
   >      T1["T1 #1206"] --> integ
   >      T2["T2 #1207 (stacked)"] --> T1
   >    ```
   > Label every node with **task id + what it does + its PR number** once it exists —
   > the same identity rule as everywhere else. Draw them **once**, from the plan; a
   > diagram is structure, not status, so it does **not** get re-rendered as PRs move
   > (GitHub already shows live status next to each referenced PR). Re-render only if
   > the plan itself changes — a task added, dropped, or re-based. Do NOT mirror PR
   status / CI / merge state into the body and do NOT keep re-editing it: GitHub
   already renders the live status of referenced PRs, so re-writing the description on
   every change is wasted churn (and re-triggers noise). Record
   `planPrNumber` / `planPrUrl`, and seed `prTitlePattern` from the title you used.

   > **Never create labels, and never label a PR.** The repo's label set belongs to
   > the repo's humans, not to a tool passing through. Cadence does not run
   > `gh label create`, does not `--add-label`, and does not ask for one to be made.
   > Task↔PR identity is carried by the identity header, the cycle map, and the
   > naming rules — see **Task ↔ PR identity**.
   This PR stays a **draft** until every task has merged into integration (step 3
   un-drafts it), and the **human** merges it last.
5. **Tell the user where everything lives — and what the merge policy is — once, at run
   open.** Print the run dir (absolute), the plan doc, the integration branch + plan PR
   URL, the cycle label, `<runDir>/report.md` (renderable any time with
   `/cadence:report`), and **one explicit line on who merges what** — e.g. "task PRs
   merge automatically once you approve them and every guard holds; the plan PR into
   `main` is always yours" (or "merges are human-only" when
   `approvalMergePolicy: "off"`). That line is a promise for the life of the run: it is
   pinned in `run.json` and an upgrade will not rewrite it. This is the only time it's spelled out in full; after that it's
   the one-line footer on every turn.
6. Compute the first dispatch set = tasks whose **base is available**. Initially that
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
| `pending` (base available) | **spec agent** — verify-and-extend the brief, decides `complexity`; **fuses into implement for `trivial` only** (**opus/high**) → `specified` (or `open` when fused) |
| `specified` | **implement agent** — TDD build → opens PR (**model = `modelPolicy[complexity]`**) → `open` |
| `open` **with a snapshot delta** | **monitor agent** — Monitor pass on the settled PR (**sonnet/low**) |
| `open`, no delta, but **readiness or auto-merge may now pass** (draft whose blocker just merged, approval recorded and guards satisfiable, a base that advanced) | **monitor agent** (**sonnet/low**) — the readiness / auto-merge / base-sync re-check is work even when the PR itself didn't change |
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

**Log a `task.spawn` event for EVERY agent you spawn** — spec, implement, monitor, fix,
cleanup, and every ad-hoc one (repair, verification, housekeeping, reconciliation)
— with `agentKind`, `model`, and effort. A spawn with no event did not happen as far as
the report is concerned, and the cost section then reads `not captured` while the run
was in fact spawning dozens of agents. There is no such thing as an unlogged spawn.

> #### STATE THE RULE, NEVER THE DERIVED NUMBER (brief-writing invariant)
> A brief may carry facts you can point at — a path, a requirement id, a blocker's PR
> number, the plan's own words. It **must not carry a count, a total, or an arithmetic
> claim you computed yourself.** Write the *rule* and let the agent, which is standing
> in the worktree, do the counting.
>
> This is not hypothetical. In one production cycle the orchestrator put three derived
> numbers into briefs and all three were wrong: a touch set "going 10 → 11 files" that
> was a double count; a claim about which file a script generates that was taken from
> another agent's report and never verified; and `REQUIRED_PATHS` "goes 21 → 23", which
> confused OpenAPI **path keys** with **operations** — the task added two GET methods
> to path items that already existed, so it added **zero** entries. An agent trusting
> that number would have padded the array to 23 with paths that do not exist and
> shipped a false docblock. All three were caught only because the receiving agents
> counted for themselves against the live tree instead of believing the brief.
>
> - ❌ "`REQUIRED_PATHS` goes from 21 to 23." · ❌ "This touches 11 files."
>   ❌ "`CLAUDE.md` is generated by `build-agents.ts`."
> - ✅ "`REQUIRED_PATHS` must contain exactly one entry per OpenAPI **path key** this
>   task exposes — count the live array yourself and say what you found; adding a
>   method to an existing path item adds no entry."
> - ✅ "Touch only the files your brief's touch set justifies; report the actual count
>   in your PR body." · ✅ "Check whether `CLAUDE.md` is generated before editing it —
>   `build-agents.ts` writes a specific set of artifacts; verify which."
>
> If you *must* pass on a number (from a plan, a prior agent, a report), label it
> **unverified** and name its source, and instruct the agent to verify before relying
> on it. An agent that finds a brief's claim false should **decline and report it** —
> that is the correct behaviour, not insubordination; record it as an `incident` with
> `kind: "brief.false-premise"` and correct the brief.

**Before spawning a `pending` task's first agent, check nobody else already took it.**
Two orchestrators on one task is how duplicate PRs and lost work happen, and in
production the only reason a collision was caught was that the other session's agent
happened to relay through this one. So, in the same read-only pass: if the task's
branch already exists on the remote, or a PR matching `prTitlePattern` for this task id
is already open, **do not spawn**. Record it in `attention` as
`possible-concurrent-session` with the branch/PR it found, log an `incident`
(`kind: "task.collision"`), and let the user decide who owns the task. Adopting the
existing branch, opening a second PR, or force-pushing over it are all wrong: the other
session may be mid-write, and neither of you can see the other's state.

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
- **Sync, don't block, when a base advances:** when a blocker's branch gets new commits
  or merges, the dependent task's agent **merges the updated base in** (never a rebase,
  never a force-push on an open PR) — or refreshes/retires its join branch — on its next
  tick (Monitor pass step 4), then re-checks that the diff still shows only its own
  scope. Flow continues, nothing halts.
- **Un-draft, and merge on approval, without being asked:** each tick the task agents
  re-run the readiness checklist and the auto-merge guards themselves. The
  orchestrator only records the outcome and reports it.
- **Plan PR:** when every task PR is merged into integration, handle the plan PR
  (un-draft, parent tracker → In Review; see "Plan-PR handling"). The plan PR is
  itself driven by a per-task-style agent in the integration worktree when it has
  CI/comments.
- **Append this wake-up's `tick` event — EVERY wake-up, quiet ones included.** One line
  to `events/run.jsonl`: `{ts, actor: "orchestrator", kind: "tick", …}` with `spawns`
  (how many, by kind), `deltas` (which tasks had news), `quiet` (true/false),
  `quietTicks`, and the `sleptSeconds` you are about to schedule. This is the only
  record that a wake-up ever happened: a multi-day run that logs none of them produces a
  report whose entire cost section reads `not captured` — which is exactly what happened
  in production, where the run could not say how many times it woke, how many ticks were
  quiet, or whether the backoff was working. A tick costs one JSON line; skipping it
  costs the whole Cost section. Write it, then re-arm.
- **Re-arm the loop (mandatory unless fully done) — at the ADAPTIVE interval.**
  Call **ScheduleWakeup** with `delaySeconds` computed from `run.json.monitorBackoff`
  (`{baseSeconds: 180, maxSeconds: 1800, quietTicks}`):
  - **Any agent in flight, any spawn this tick, any snapshot delta, new work
    dispatchable, a base ref that moved, a PR that is conflicted/behind, or a
    `mergeable: UNKNOWN` still unresolved** → the run is HOT: reset `quietTicks = 0`,
    sleep `baseSeconds`. A merge the human just made must never be followed by a
    30-minute silence while dependents sit conflicted.
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
  is `done`/`failed` — remove the integration worktree/branch, **render
  `<runDir>/report.md`** from the event logs (see Evidence capture), write the final
  summary **leading with the report's absolute path**, send the one completion
  notification, and **omit ScheduleWakeup**. The run is complete.

## Per-task orchestrator agent (summary — full playbook in references/task-agent.md)
The top orchestrator spawns this agent for an **idle** task with work. It reads
`references/task-agent.md` first and follows it exactly. In brief: it resumes from
its `tasks/<id>.json` and does only the step its `status` calls for — **Spec**
(worktree + descriptive branch off its base, **building its join branch first when it
has 2+ blockers**; verify-and-extend the plan brief, graphify-first; decide
`complexity`; raise any **open decision** it can't settle; fused straight into
implement for `trivial` only) → **Implement** (TDD; green gate; complexity-scaled
pre-push self-review — run EXACTLY ONCE, only the lint/tests gate is re-run after
fixes, never a second `/code-review`; open its own PR against its `baseBranch`, then
run the **readiness checklist** and un-draft itself if it passes) →
**Monitor pass** (ONE GraphQL fetch of all signals; JUDGE BEFORE YOU ACT on every
comment/review; NO SILENT FIXES — verified `gh` replies with captured URLs; SIGNAL
RESOLUTION — resolve fixed threads + re-request the reviewer, bounded by the
REVIEW CONVERGENCE BOUND: max 3 fix rounds per reviewer, then a summary comment
and park for the human; harvest answers to its open decisions; CI fixes; base syncs /
join refreshes onto its advancing base; re-run readiness; and **merge itself under an
intact human approval** per Approval-authorized auto-merge) → **Cleanup** on merge,
then it dies. It is the sole writer of its `tasks/<id>.json`, syncs its own tracker
issue on every transition, and never touches `main`, the plan PR's merge button, or
another task's branch/PR.

## Evidence capture + the cycle report

A cycle runs for days across sessions, so **what happened must be written down as it
happens** — at the end there is no memory to reconstruct it from. Both levels append to
their own append-only log (same single-writer rule as the state files):
`events/run.jsonl` (yours) and `events/<id>.jsonl` (each task agent's). One JSON line per
event: `{ts, actor, kind, detail, pr?, url?, sha?, model?}`.

**`run.json` and `tasks/<id>.json` are overwritten in place — they say what is true
now, never what happened. The event log is the ledger, and it is the only one.** So:
**if you wrote a field, you owe an event.** Every task `status` transition emits
`state.changed` (`from` → `to` + trigger); every PR snapshot field you observe changing
emits `snapshot.delta` (which fields, `old` → `new`) — that is the per-PR change ledger
the report renders; every base ref that moves emits `base.moved` with the dependents it
affects.

**The rule binds hardest for the things you'd rather not write down.** In production the
event ledger ended up *thinner* than the state files: a run whose `run.json` recorded
three of the orchestrator's own errors, a cross-session collision, an open risk, and an
answered decision had **no event line for any of them** — the reconstruction had to come
out of state, which is the opposite of the design. Every anomaly gets an `incident`
event as it happens (`kind`: `brief.false-premise` · `topology.skew` · `task.collision` ·
`orchestrator.error` · `external.merge` · whatever it actually was), with what happened,
what caught it, and what was done. **Cadence's own mistakes are first-class evidence**;
they are the most valuable lines in the log, they are what makes the next version
better, and a run that hides them has broken the one rule the report depends on. Before
you write any field to `run.json` that is not a routine pointer — an incident, a stall,
an unresolved decision, an attention entry with a cause — write its event first.

Log the moment it happens — spawns, complexity decisions, PR opened/ready/re-drafted,
decisions raised/answered, reviews and fix rounds, CI flips, base syncs, conflicts and
how they were resolved, silent re-targets, scope checks, joins, guard blocks,
auto-merges, merges, stalls, escalations, failed gates, and one `tick` line per
wake-up. The full kind list and the rendering rules are in
**`references/cycle-report.md`**.

From those logs Cadence renders **`<runDir>/report.md`** — a single self-contained file
the human can read or paste to anyone: outcome, a per-task table, a **per-task ledger** (every state and
PR change with its timestamp and evidence, including conflict latency: base moved →
detected → clean again), **what needed you** (with how long each thing waited), **what
went wrong** (Cadence's own failures included), flow health, cost, timeline, follow-ups,
and a copy-pasteable *Feedback for the Cadence maintainer* section.

- **Write it at the end condition, before the final summary**, and print its **absolute
  path** — first line of the summary, not buried at the bottom.
- **Render it on demand** whenever the user asks how the cycle went (or runs
  `/cadence:report`), marked `IN FLIGHT`. Say the path every time you write or refresh
  it — "written to <path>", never just "the report is ready".
- **Its location is never a mystery:** announced in full at run open, then carried as a
  one-line footer on every turn (see the Turn-end invariant), so the user never has to
  ask where it is or dig through `.cadence/`.
- **Never fabricate a metric** — an event that wasn't logged is `not captured`, never
  estimated. And never soften the failure sections: a report whose "what went wrong" is
  empty after a run with re-drafts, stalls, or parked reviewers is a broken report.

## Task ↔ PR identity (the human must NEVER have to ask which PR is which task)

A cycle opens 5–12 PRs at once. If the mapping between a task and its PR lives only in
the plan doc and the state files, the human ends up asking "which PR was T3 again?" on
every single turn — a defect, and an easy one to kill: **stamp the identity into every
surface where a PR appears.** Three places, all mandatory — and **not one of them is a
label.** Cadence never creates a label and never labels a PR: the repo's label set
belongs to its humans, and a tool passing through for a few days does not get to write
to it. Identity rides in what Cadence is already writing anyway:

1. **The PR body's first line — an identity header, in BOTH body sizes.** Not just the
   full template; a two-sentence simple PR gets it too:
   ```
   **T2 · Wire the matcher into the inbound pipeline** — cycle `reply-followups` ·
   plan PR #1200 · ABC-1234 · base `cadence/reply-followups-t1-add-reply-matcher`
   (stacked on #1206 — merge after it)
   ```
   Task id **and** a plain-language title, the cycle slug, a link back to the plan PR,
   the tracker key if any, and what it's based on. Omit tokens that don't exist; never
   invent a ticket key.
2. **The plan PR body is the cycle's canonical map** — a table, appended one row per
   task as its PR opens (still write-once per row; don't re-edit rows afterwards):

   | Task | What it does | PR | Base |
   |---|---|---|---|
   | T1 | Add the reply-correlation matcher | #1206 | integration |
   | T2 | Wire the matcher into the inbound pipeline | #1207 | #1206 (stacked) |

   GitHub renders each referenced PR's live state next to it, so this one table
   answers "what's the state of the cycle?" without touching the description again.
3. **Every message you write** — turn summaries, `attention` entries, dispatch notes,
   PR comments that mention sibling work. The rule is absolute: **a task id never
   appears without its PR number and a plain-language description, and a PR number
   never appears without what it does.** `T3` alone is meaningless to the human;
   `#1208 (backfill existing correlations)` is self-explanatory. This applies to the
   per-task agents too — the "be didactic about references" rule in
   `references/task-agent.md` is the same rule seen from the PR side.

PR **titles** carry the task's real work in plain words (they're derived from the task
title via the PR title convention), so a title plus the identity header means a reader
landing cold on any PR knows what it is, which task it came from, and where it sits in
the cycle.

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
2. Local gate green (lint/format/tests) **and CI TERMINAL AND GREEN on this exact
   head SHA** — see below. "No failing check" is not the test.
3. Self-review done for its complexity (or `trivial` → not required).
4. No unanswered **blocking** open decision.
5. PR body complete: what & why, how to test, decision log if any, open decisions if any.
6. **Current with its base and clean in scope** — not `DIRTY`/`BEHIND`/`CONFLICTING`,
   and the diff against its base contains only this task's own work (the **scope
   check**, below). Inviting review onto a conflicted PR, or onto one showing another
   task's files, wastes the reviewer's time and is what makes a PR "not worth reviewing
   yet".

If a PR later regresses (new in-flight work, a new blocking decision, CI goes red),
re-draft it (`gh pr ready --undo`) and post a one-line comment saying why.

> #### Item 2 in full — CI must be TERMINAL and GREEN on THIS SHA
> The wording used to be "CI has no failing check", and it let two PRs un-draft with
> no green run behind them: one where the checks had **not been created yet** ("no CI
> checks report on this branch" — an *empty* check set trivially has no failing
> check), and one where two checks were still `QUEUED` — queued is not failing. Both
> were invitations to review code nothing had verified. So the test is positive, not
> negative:
> - the rollup is on the **current `headRefOid`** — a green run against the previous
>   SHA says nothing about this one;
> - the check set is **non-empty**, and every check has reached a **terminal**
>   conclusion (`SUCCESS` / `NEUTRAL` / `SKIPPED`) — nothing `QUEUED`, `IN_PROGRESS`,
>   `PENDING`, `EXPECTED`, or `WAITING`;
> - no check is `FAILURE` / `TIMED_OUT` / `CANCELLED` / `ACTION_REQUIRED`.
>
> Anything else → **item 2 is false**: stay draft with `draftReason: "ci_pending"`
> and re-evaluate next tick. This is not a stall — the tick that sees the rollup go
> terminal-green un-drafts the PR without anyone asking.
>
> **The one exception, and it must be proven, not assumed:** a repo with no CI at all.
> Confirm it — the repo has no workflows that trigger on this PR's events — record
> `ciExpected: false` in the task file with that reason, and the local gate carries
> readiness alone. An empty check set is otherwise treated as **not yet reported**,
> never as green.

### Staying current with a moving base — and keeping the diff honest

The human merges PRs while the cycle runs. That is normal and expected, and it is the
moment most likely to leave a PR conflicted with a diff that no longer matches its
scope. Two rules, both executed by the task's own agent (mechanics in
`references/task-agent.md`, Monitor step 4):

- **Merge the base in — never force-push.** When a base advances, bring the branch
  current with `gh api repos/{owner}/{repo}/pulls/<n>/update-branch -X PUT` (GitHub's
  own merge-base-in, no local work) or a local `git merge origin/<base>` when that
  fails or conflicts. **Do not rebase and do not force-push an open PR**: the human may
  be mid-review, and a rewrite unanchors their inline comments and can dismiss their
  approval. A merge commit is a small price for review continuity.
- **Then check the scope.** `git diff origin/<base>...HEAD --name-only` must contain
  only files this task legitimately touches (its plan brief's touch set + anything the
  spec phase justified). If foreign files appear, the sync went wrong — do not invite
  review on it. The usual cause is a **squash-merged blocker**: the blocker's work
  landed on the base as one new commit while this branch still carries the blocker's
  *original* commits, so the same change exists twice — git conflicts, and the PR shows
  the blocker's files as if they were this task's. The fix is to resolve those conflicts
  **in the base's favor** (the blocker's version already merged; this branch's copy is a
  stale duplicate), then re-run the check until the diff is this task's work alone.

Record `filesChanged` and the scope-check verdict in the **task file** — that's where
current-state belongs. The **PR body gets a scope statement in words** ("touches
`libs/storage/**` and its registration; nothing else"), never a count: GitHub renders the
count live, and a number written into a description is wrong one commit later (see
**DURABLE, NOT VOLATILE** in `references/pr-template.md`). An unexplained sprawling diff
for a small task is a defect, not a big task — so if the diff legitimately reaches
further, the body says *why*.

### Approval-authorized auto-merge (task PRs only — NEVER the plan PR)
**An approving review by a human on a task PR is that human's authorization to merge
it.** Waiting for them to also click the button just parks finished work. So the
task's own agent merges it — *if and only if* every guard below holds **in the same
monitor tick that merges**, verified from that tick's GraphQL payload and the local
git state. **Read `run.json.approvalMergePolicy` — never the default in this file.** It
is pinned at run open: `"auto"` for runs opened under a version that has this feature,
`"off"` for any run that predates it or whose user asked for the click. A run that began
under a human-only policy stays human-only until *the user* says otherwise, no matter
what version is installed now (see Pin the policy). Record the choice when they do.

| # | Guard | Fails if… |
|---|---|---|
| 1 | **Human approval exists** | the only approvals are from bots/apps — a bot approval never authorizes a merge on its own |
| 2 | **Approval still stands** | it was dismissed, or any *later* review is `CHANGES_REQUESTED`, or `reviewDecision ≠ APPROVED` |
| 3 | **Approval still describes the code** | the head moved since `approvedSha` in a way that is more than a base-sync merge or a `trivial`-class edit (see below) |
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
- a **base-sync merge** — the only new commits are the merge of the updated base (plus
  conflict resolutions taking the base's already-merged version), and the branch's own
  patch set against its merge base is unchanged: `git diff <mergeBase>..HEAD` yields the
  same patch-ids as it did at `approvedSha`; or
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
   at which SHA, what changed since (nothing / base-sync / trivial), and the guards that
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
- **Never mirror live state into any PR body — this is the general rule, not a plan-PR
  quirk.** File counts, test counts, CI status, review state, "N of M merged": GitHub
  renders all of it live, and a description that restates it is stale on the next
  commit. Bodies carry intent; the tabs carry status. See **DURABLE, NOT VOLATILE** in
  `references/pr-template.md`.
- **The plan PR body opens with the cycle plan diagrams** (waves + dependencies, and
  branch topology) — see step 1.4. They are drawn once from the plan and only redrawn if
  the plan itself changes; they show structure, not status.
- **Add a task's row to the plan PR's cycle map once, when its PR first opens**
  (`| T-id | <plain-language what it does> | #<prNumber> | <base> |` — see **Task ↔ PR
  identity**). After that, leave the body alone — don't tick, restyle, or re-edit rows
  as CI runs or PRs merge. GitHub auto-renders the live status of every referenced PR,
  so churning the description each transition wastes time and adds noise. (Completion
  is signalled by the one-time un-draft + comment below, not by editing the body.)
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
