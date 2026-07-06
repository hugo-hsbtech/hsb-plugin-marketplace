---
name: cycle-executor
description: >
  Autonomous project-manager that EXECUTES a parallel cycle plan (from
  /hsb:planner or any wave schedule). For each unblocked task it runs the full
  delivery lifecycle: git worktree → branch → superpowers spec/plan/implement
  (auto-approving every human-in-the-loop gate by following the recommended
  option and logging the alternatives) → open a PR with UAT, mermaid diagrams,
  and a decision log → then MONITOR the PR (comments, reviews, CI, conflicts) on
  a scheduled-wakeup loop, replying to and fixing issues until a HUMAN merges it.
  On merge it destroys the worktree + branch and retires that task. Dispatches
  all tasks as parallel local subagents in isolated worktrees and FLOWS end to end
  — it never freezes a task waiting for another's PR to merge. Dependencies are
  expressed by PR base, not by waiting: a task with ONE blocker stacks its PR on
  that blocker's branch; a task with TWO OR MORE blockers targets the integration
  branch (the convergence point). One integration branch holds the plan docs as a
  plan PR → main; the human merges the plan PR last.
  Use when the user says: "execute the cycles", "ship the planned cycles", "run
  the waves", "implement these tasks autonomously", "drive these PRs", or runs the
  /hsb:execute-cycles command (which loads this skill).
  When tasks are linked to Linear/Jira/etc., it mirrors each task's status into the
  tracker as it executes (In Progress → In Review → Done/Blocked).
  HARD RULES: one dedicated agent and one PR per task (never combine tasks into a
  shared branch/PR); a task PR targets its single blocker's branch, or the
  integration branch when it has zero or 2+ blockers — never main; never push or
  commit to main; by default never merge a PR autonomously (the human merges) — but
  if the user explicitly authorizes merging for a task/wave/cycle, honor it; keep
  any not-yet-mergeable PR as a DRAFT so a human can't merge it mid-flight; keep
  monitoring while any PR is open.
---

# Execute Cycles — Autonomous Delivery PM

Take a **cycle plan** (waves of parallel-safe tasks from `/hsb:planner`) and drive
each task to a merge-ready PR, fully autonomously except the merge itself. You are the
PM: you dispatch task agents, you keep PRs healthy, you don't merge (unless the user
tells you to), and you never touch `main`. **The process FLOWS end to end — it never
freezes a task waiting for another's PR to merge.**

**Branch topology — flow via PR base, not merge gates (read first).** A cycle has ONE
**integration branch** (`hsb/<slug>-integration`) that holds the plan/spec docs and is
itself a **plan PR → `main`** (opened as draft). Dependencies are expressed by where a
task's PR is **based**, so work flows without waiting for merges:
- **No blocker** → branch off, and PR targets, the **integration branch**.
- **Exactly one blocker** → branch off, and PR **stacks on, that blocker's branch**
  (`--base hsb/<slug>-t<blockerId>-<slug>`). The task starts as soon as the blocker's
  branch exists — no need to wait for it to merge.
- **Two or more blockers** → can't stack on several at once, so branch off and target
  the **integration branch** (the convergence point where all blockers land). Rebase
  as blockers merge in. This multi-parent join is the only place ordering is enforced.

No task PR ever targets `main`. Once all tasks land in integration, the single plan PR
is the human's one clean merge into `main`.

```
main
 └─ hsb/<slug>-integration                       ← plan PR (plan docs) → main   [human merges LAST]
     ├─ hsb/<slug>-t1-add-reply-matcher          ← no blocker  → PR base: integration
     │   └─ hsb/<slug>-t2-wire-inbound-pipeline   ← needs t1    → PR base: t1's branch (stacked)
     └─ hsb/<slug>-t3-backfill-correlations       ← needs t1+t2 → PR base: integration (2+ blockers)
```
(Task branches are **descriptive**: `hsb/<slug>-t<id>-<task-slug>`, the slug says
what the task does.)

## Non-negotiable rules
- **Never push or commit to `main`.** Task work happens on a per-task branch inside
  an isolated worktree, branched from its **base** — the integration branch, or a
  single blocker's branch when stacked (see Branch topology) — never from `main`.
  (One exception, by design: small **plan-PR** review/CI fixes are committed
  **directly to the integration branch** — it's a feature branch, not `main` —
  instead of via a child PR. See Plan-PR handling.)
- **A task PR's base is chosen by its blockers, and is NEVER `main`.** Zero or 2+
  blockers → base = the **integration branch**; exactly one blocker → base = **that
  blocker's branch** (stacked). Only the single **plan PR** targets `main`. A task
  PR opened against `main` is a bug — re-target it (`gh pr edit <n> --base <base>`).
- **Flow, don't gate on merges.** Never freeze a task waiting for another task's PR
  to merge. A stacked task starts as soon as its blocker's branch exists; a
  multi-blocker task targets integration and rebases as blockers land. The only
  ordering that exists is which branch a PR is based on.
- **By default, never merge a PR autonomously — but honor an explicit user override.**
  Merging is normally the human's gate, for **both** task PRs and the plan PR; you
  monitor and fix until *they* merge. This is overridable ONLY by an explicit user
  instruction ("you may merge wave 1", "auto-merge this cycle when green"); when the
  user grants it, honor it for exactly the scope they named, record the authorization
  (who/what/when) in `run.json` and the PR Decision Log, and still never merge a red
  or draft PR. Absent such an instruction, do not merge.
- **Keep a PR that isn't meant to be merged as a DRAFT.** A PR is `--draft` until it
  is genuinely ready AND safe for a human to merge right now — i.e. its agent has no
  work in flight, CI is green, and (if stacked) its base has merged. Marking a PR
  "ready for review" is a signal that a human may merge it; never send that signal
  while a subagent is still working the PR or while it's stacked on an unmerged base,
  or a human may merge it mid-flight and force a re-do. Un-draft (`gh pr ready`) only
  when settled; if new in-flight work starts, convert back to draft (`gh pr edit
  <n> --add-draft` / `gh pr ready --undo`) and say so in a comment.
- **Monitoring is mandatory and self-sustaining while a PR is open.** Opening a PR
  is NOT "done" — the job is done only when every PR is **merged + cleaned up**.
  You keep re-checking (comments, reviews, CI, conflicts) and resolving issues on a
  schedule until merge. See **Turn-end invariant** — you may not stop with an open
  PR and no scheduled re-entry.
- **One dedicated agent per task.** Every task in a wave is dispatched as its OWN
  parallel subagent in its OWN worktree. Never collapse multiple tasks into one
  agent, and never implement a task inline yourself. See dispatch step 2.
- **Branch names must describe the work.** Every task branch is
  `hsb/<slug>-t<id>-<task-slug>`, where `<task-slug>` is a 2–5-word kebab-case summary
  of what the task actually does (derived from its title/goal during the Spec phase).
  A bare `hsb/<slug>-t<id>` that doesn't say what the PR does is not acceptable.
- **One PR per task — the default, always.** Each task ships on its own branch in
  its own worktree and opens its **own** PR. Never combine multiple tasks' changes
  into a single shared branch or PR. The only exception: a task whose change is so
  trivial it isn't worth a standalone PR (e.g. a one-line tweak) — it may be folded
  into a closely-related task's PR, but only with its own clearly-labeled section
  (task id, what/why, decision log) in that PR body, and the fold noted in state.
  When unsure, open a separate PR.
- **Preflight gate before any work.** `gh` must be authenticated as the correct
  user AND every required MCP server must be connected — verified and recorded in
  state *before* a single task is dispatched. Abort the run if the gate fails. All
  PR replies use real `gh` commands.
- **Judge reviews, don't obey them.** Comments and reviews are *suggestions to
  evaluate*, never commands to implement blindly. For each one, verify it against the
  code and decide if it's correct, valuable, reasonable, and in-scope — then agree,
  propose a better alternative, decline with a reasoned reply, or ask for
  clarification. A wrong or out-of-scope suggestion is declined (with respect and
  evidence), not implemented. See **Monitor pass → JUDGE BEFORE YOU ACT**.
- **No silent fixes.** Whenever you change code in response to a review comment, a
  review, red CI, or a conflict rebase, you MUST post a real `gh` reply/comment on
  the PR *and verify it posted* (capture its URL) before treating the item as
  handled. A pushed commit is never a substitute for a reply. Every item — whether
  you implemented it or declined it — gets a posted, reasoned reply. See **Monitor
  pass → NO SILENT FIXES**.
- **Mirror task status to the issue tracker.** If a task (or the cycle) is linked to
  Linear / Jira / GitHub Issues / Asana / any tracker, every internal status
  transition is reflected back to that tool's issue automatically — moving the issue
  through its workflow states and commenting the PR link — so the tracker is always
  the truth a human sees. See **Issue-tracker status sync**. Never leave a linked
  issue stale while its task moves.
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
  its alternatives + rollback) in the PR Decision Log, and continue. The user
  steers *asynchronously* — by replying here to the orchestrator, or via PR
  comments/reviews (which the monitor pass picks up and applies). The **only**
  reasons to stop and surface to the user are hard blockers that make autonomous
  work impossible: no/invalid plan, `gh` not authenticated as the right user, or a
  task whose gate can't go green (left as a draft PR with a blocker note). Anything
  decidable, you decide.

## Turn-end invariant (READ THIS BEFORE ENDING ANY TURN)
The single most common failure is stopping after PRs are opened and never coming
back to monitor. To make that impossible, end **every** turn by checking state:

> **If ANY task is `status ∈ {pending, specifying, specified, implementing, open, fixing}` OR the plan PR
> is not yet merged into `main`, the LAST action of this turn MUST be a
> `ScheduleWakeup` call** (re-entering with the same `/hsb:execute-cycles
> <plan-path>`). Only when every task is `done`/`failed` **and the plan PR has been
> merged** may you end without scheduling — and then you write the final summary.

You are never "finished" because you opened PRs, nor because all task PRs merged —
the plan PR into `main` is the last gate. You are finished only when the task files
show no active task **and** the plan PR is merged. If you are about to produce a
closing message while any task PR or the plan PR is still open, STOP and call
`ScheduleWakeup` instead.

**Also before ending any turn:** run the NO SILENT FIXES assertion — every comment/
review/CI/conflict you acted on this turn must have a recorded `replyUrl` on the PR.
Never end a turn having pushed a fix without posting (and verifying) its reply.

## Inputs
- A cycle plan: a path to a cycle-plan markdown produced by `/hsb:planner` — named
  `docs/plans/proposed/<YYYYMMDD-HHMM>-<slug-of-proposed>-<task-id>.md` — or the wave
  schedule already in context. If none is given, run `/hsb:planner` first or ask for
  the plan. Do not invent tasks.
- Read the plan's **metadata header** for the canonical `Slug` and `Task-id` — use
  that `slug` for all `hsb/<slug>-*` branches and the cycle state directory (see State
  directory). Do NOT derive the slug by parsing the filename (the name is timestamped).
- Parse: waves, per-task IDs, summaries, dependencies, and per-task context briefs
  (touch sets, acceptance criteria).

## State directory (survives across wakeups & sessions; split by owner)
Monitoring spans hours-to-days and re-enters via scheduled wakeups, so persist
everything durably under `.hsb/cycles/` at repo root (create `.hsb/` and add it to
the repo's `.gitignore` if missing — never commit cycle state onto a feature branch).
Schema in `references/execution-state.md`.

**The state is a per-run directory, NOT a single file** — so each task's agent can own
and write its own state without racing the others (parallel agents writing one shared
JSON would clobber each other):
```
.hsb/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/
  run.json            ← ORCHESTRATOR-owned: slug, planPath, integrationBranch,
                         planPr*, prTitlePattern, preflight, wave schedule,
                         issueTracker, monitorIntervalSeconds, nextWakeupAt
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
1. Glob `.hsb/cycles/*-<slug>-cycle/run.json`.
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
  branch + plan PR; own `run.json`; each wakeup, spawn a per-task agent only for
  every *idle* active task (one with **no agent in flight** — see Idle-gating);
  gate/dispatch waves as blockers merge; manage the plan PR; call `ScheduleWakeup`.
  It reads `tasks/<id>.json` to learn status but never writes them. Keep its own
  console output minimal — the work belongs in the agents.
- **Per-task orchestrator agent (one Agent per idle active task):** resumes from its
  `tasks/<id>.json` and drives ITS task one step as far as it can right now — spec
  (analysis/plan) → implement→PR if pre-PR, else monitor→fix→reply, else cleanup if
  merged. It may spawn its own sub-agents (TDD/analysis via superpowers). It is the
  **sole writer of its `tasks/<id>.json`**
  and the one that does all `gh` work for its PR. It **dies when its PR is merged**
  (after cleanup); the top orchestrator simply stops re-spawning it. Continuity across
  ticks is the worktree + task file, not the process.

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
- **Only spawn a monitor tick for a task whose status is `open` with no agent in
  flight** (settled, PR up, awaiting human). `pending` → spawn a **spec** agent;
  `specified` → spawn an **implement** agent; `merged` → spawn a cleanup agent;
  `specifying`/`implementing`/`fixing` mean an agent is already in flight → skip.
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
> task's agent every tick. Each agent invocation = "advance my task as far as
> possible now, write my state, return."

### Model selection (by PHASE — analysis is always Opus; implementation by complexity)
Spawned agents must NOT all inherit the orchestrator's Opus. The model is chosen by
**what the agent is doing**, and complexity is an **output of the analysis phase**,
not something assigned when the task was defined:

| Phase / agent kind | What it does | Model + effort (`modelPolicy`) |
|---|---|---|
| **spec** (analysis / planning / specifying) | superpowers `brainstorming` + `writing-plans`, real codebase analysis in context; **decides this task's `complexity`** | **`opus`, effort `high`** — always, every task |
| **implement** — `complexity: high` | TDD build of the most complex tasks | **`opus`, effort `medium`** |
| **implement** — `complexity: medium`/`low`/`trivial` | TDD build of lighter tasks | **`sonnet`** (low effort for `low`/`trivial`) |
| **monitor / fix / cleanup** | read PR state, reply, small fixes, worktree teardown — light | **`sonnet`**, effort `low` |

Rules:
- **Any analysis/planning/specifying done via a subagent runs on `opus`, high
  effort** — that's where correctness is won. This includes the per-task spec agent
  AND any sub-subagents it spawns to analyze the codebase.
- **Complexity is set during the spec phase** (the spec agent writes `complexity` to
  its `tasks/<id>.json` as a finding), NOT pre-assigned by the planner. The
  *implement* agent is then spawned at `modelPolicy[complexity]`.
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
is set may step 1 run. Re-verify the gate at the start of any wakeup that is about
to dispatch a new wave (auth/MCP can drop between sessions).

1. **`gh` authenticated as the correct user.** Run `gh auth status`. Confirm it
   reports `Logged in to github.com` and capture the account login. If it errors, is
   logged out, or is the *wrong* account for this repo, STOP (`gh auth login` /
   `gh auth switch`) — PR creation and comment replies would otherwise fail silently
   or post as the wrong identity. Record `preflight.ghAuth = "ok"` and `ghLogin`.
2. **Repo target locked.** `gh repo view --json nameWithOwner`; record `repo`.
3. **Required MCP servers connected & authenticated.** Determine which MCP servers
   the run depends on (e.g. the Linear/Jira server when the plan/tasks come from or
   report to an issue tracker, plus any other server a task brief names). For each,
   do a cheap read to prove it is reachable and authed (e.g. a list/whoami call). If
   a required server is missing or unauthenticated, STOP and tell the user which
   server to connect/authenticate. Record each as `preflight.mcp[server] = "ok"`.
   (If the run genuinely needs no MCP server, record `preflight.mcp = "none"`.)
4. **Issue-tracker linkage (if the plan references one).** If the plan/tasks carry
   tracker keys (Linear/Jira/GitHub Issues/etc.), the tracker's MCP server is
   **required** (not optional) and must have **write** access — confirm you can
   update an issue (read its workflow states; a dry capability check). Map each task
   to its issue key/id/url and record under `issueTracker` + per-task `issue`. Also
   discover and cache the project's **workflow state names** so the sync maps to real
   states, not guesses. If linked but the tracker is read-only/unreachable, STOP.
5. **Stamp the gate.** Only when 1–4 all pass, set `preflight.passedAt` and proceed.

Monitoring uses the in-session `ScheduleWakeup` loop only (no cron). It is cheap and
always runs with your verified auth; it pauses if the session is fully terminated
and resumes when you re-run `/hsb:execute-cycles <plan-path>`.

### 1. Plan the run + open the integration (plan) PR
1. Load the plan; write/refresh `run.json` with the task roster + wave schedule
   (every task `pending`) and the default **`modelPolicy`** (see Model selection).
   **Do NOT set `complexity` here** — it is determined later by each task's Spec
   phase and written by that agent. Per-task `tasks/<id>.json` files are created by
   their own agents on first spawn — the orchestrator never pre-writes them.
2. **Create the integration branch as a worktree off latest `main`:**
   `git fetch origin && git worktree add .claude/worktrees/hsb-<slug>-integration -b
   hsb/<slug>-integration origin/main`. Record `integrationBranch` +
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
      `git push -u origin hsb/<slug>-integration`.
   4. **Post-condition — assert `main` is clean of cycle files.** Run
      `git status --porcelain` on the `main` checkout and confirm **none of the in-scope
      doc paths remain** there; they now live only on the integration branch. Out-of-
      scope uncommitted changes are expected to remain and are noted **once** in the run
      summary, never committed. If any in-scope doc is still dangling on `main`, move +
      commit it before continuing — **never open the plan PR with cycle docs left
      uncommitted on `main`.**
4. **Open the plan PR → `main` as a draft:** `gh pr create --base main --head
   hsb/<slug>-integration --draft`. Title: resolve via **PR title convention** (for
   this first PR, match the repo's house style). Body: the cycle overview + a
   **static task→PR list** — one line per task, `T-id — <title> — #<prNumber>`,
   appended **once** when that task's PR opens. Do NOT mirror PR status / CI / merge
   state into the body and do NOT keep re-editing it: GitHub already renders the live
   status of referenced PRs, so re-writing the description on every change is wasted
   churn (and re-triggers noise). Record `planPrNumber` / `planPrUrl`, and seed
   `prTitlePattern` from the title you used. This PR stays a **draft** until every
   task has merged into integration (step 3 un-drafts it), and the **human** merges
   it last.
5. Compute the first dispatch set = tasks whose **base branch exists**. Initially that
   is every task with **0 or 2+ blockers** (base = integration, which now exists) plus
   any **single-blocker** task whose blocker's branch already exists. Stacked children
   of not-yet-started blockers become active as soon as those blockers' branches are
   pushed — no merge required.

> If the run is a bare task list with no docs to commit, still create the
> integration branch and open the plan PR with just the cycle overview — it is the
> base every task PR ultimately targets.

### 2. Each tick: spawn a per-task agent only for each IDLE active task
This is the only "work" the top orchestrator dispatches — it does NOT implement,
poll, or fix anything itself. An **active** task is one not yet `done`/`failed` whose
**base branch exists** — i.e. the integration branch (for 0 or 2+ blockers) or its
single blocker's branch (for a stacked task) has been created and pushed. **This is
NOT "blockers merged"** — a stacked task is active the moment its blocker's branch
exists, so work flows without waiting for merges. Of the active tasks, act ONLY on the
**idle** ones (`agentInFlight = false`) — see Idle-gating:

| Task status (idle, no agent in flight) | Spawn this tick (model) |
|---|---|
| `pending` (base branch exists) | **spec agent** — analysis/plan, decides `complexity` (**opus/high**) → `specified` |
| `specified` | **implement agent** — TDD build → opens PR (**model = `modelPolicy[complexity]`**) → `open` |
| `open` | **monitor agent** — Monitor pass on the settled PR (**sonnet/low**) |
| `merged` | **cleanup agent** → removes worktree/branch, status `done` (**sonnet/low**) |
| `specifying` / `implementing` / `fixing` | **nothing — agent already in flight, SKIP** |

**Hard requirement: exactly one `Agent` (`general-purpose`) call per *idle active*
task, all in a single message** so they run concurrently. **Set each agent's `model`
(and effort) by PHASE from `modelPolicy`** (see Model selection): spec → opus/high,
implement → by the `complexity` the spec wrote, monitor/cleanup → the cheap `monitor`
policy. Do not let agents inherit Opus by default. Mark `agentInFlight = true`
(+ `agentKind`,
`agentStartedAt`) in `run.json` when you spawn one. Build/fix agents run in the
**background** (`run_in_background`) so a long build never blocks monitoring of other
tasks' open PRs; the quick monitor check can be foreground. Each agent resumes from
`tasks/<id>.json`; pass it the task's context brief, its `runDir`/task-file path, the
`integrationBranch`, and the current `prTitlePattern`.

Forbidden, because it breaks parallelism, isolation, one-PR-per-task, or idle-gating:
- Putting two or more tasks into one agent's prompt ("do T2 and T3").
- Sharing one branch/worktree across tasks — each task gets a distinct **descriptive**
  branch `hsb/<slug>-t<id>-<task-slug>` and worktree, so each produces a distinct PR.
- Doing a task's build/monitor/fix/state-write inline yourself instead of in its
  agent. (The top orchestrator never touches a task's PR or `tasks/<id>.json`.)
- **Spawning a monitor/second agent for a task that already has one in flight**, or
  touching that task's PR while its agent runs.
- Spawning agents one-at-a-time across separate messages (that serializes them).

Do **not** use the Agent tool's ephemeral `isolation: worktree` — it auto-cleans and
won't survive the days-long monitor. Each agent creates/reuses a **durable** git
worktree (see Spec step 1).

When an agent completes, it returns `{id, status, prNumber?, merged?, prState,
renamedTitle?, note}`. Clear its `agentInFlight` in `run.json` and record the
pointer/summary; the agent already wrote its own `tasks/<id>.json`.

### 3. Dispatch, manage the plan PR, and re-arm (the thin top loop)
After the tick's per-task agents return, the top orchestrator does only bookkeeping.
**There is no merge gate** — the loop flows:
- **Dispatch by base-branch readiness, not by merges:** a task becomes active as soon
  as its **base branch exists** (integration for 0/2+ blockers; the single blocker's
  branch for a stacked task). A stacked child is dispatched right after its parent's
  branch is pushed — it does NOT wait for the parent's PR to merge. Never freeze a
  wave waiting on another wave's merges.
- **Rebase, don't block, when a base advances:** when a blocker's branch gets new
  commits or merges, the dependent task's agent rebases onto the updated base on its
  next tick (Monitor pass step 4) — flow continues, nothing halts.
- **Plan PR:** when every task PR is merged into integration, handle the plan PR
  (un-draft, parent tracker → In Review; see "Plan-PR handling"). The plan PR is
  itself driven by a per-task-style agent in the integration worktree when it has
  CI/comments.
- **Re-arm the loop (mandatory unless fully done):** call **ScheduleWakeup** with
  `delaySeconds = 180` (quick reaction without being frenetic; the tool clamps to
  [60, 3600], so sub-60s is impossible). Use `run.json.monitorIntervalSeconds`
  (default 180). Pass `reason` naming what you're watching, and `prompt` = the same
  `/hsb:execute-cycles <plan-path>` so the next wake re-enters, locates the run, and
  continues. Record `nextWakeupAt`. This is the **last thing you do this turn**.
- **End condition:** only when the **plan PR is merged into `main`** AND every task
  is `done`/`failed` — remove the integration worktree/branch, write a final summary,
  and **omit ScheduleWakeup**. The run is complete.

## Per-task orchestrator agent (what each per-task agent does when spawned)
The top orchestrator spawns this agent for an **idle** task (no agent in flight).
**It is resumable and idempotent:** it begins by reading its `tasks/<id>.json` and
does only the step its current `status` calls for, then writes its own task file and
returns. It is the **sole writer of `tasks/<id>.json`** and the only one running `gh`
for its PR. **On every status change it also runs the Issue-tracker status sync**
(pending→In Progress, open→In Review, merged→Done, failed→Blocked) if linked.

**Resume dispatch (first thing the agent does):**
- `status = pending` (or no file yet) → set `status = specifying`, do the **Spec phase
  (steps 1–2)** → write the task's `complexity` finding + plan → end at `status =
  specified`. (Spawned on **opus/high**.)
- `status = specifying` → spec was interrupted → resume the Spec phase.
- `status = specified` → set `status = implementing`, do the **Implement phase (steps
  3–5)** → end at PR-opened, `status = open`. (Spawned at **`modelPolicy[complexity]`**.)
  **Do NOT run a Monitor pass in this round-trip** — nothing to react to on a PR you
  just opened; the first monitor is a later idle tick.
- `status = implementing` → implement was interrupted → reconcile the worktree and
  continue the Implement phase (still ending at `open`, no monitor).
- `status = open` → settled, awaiting humans/CI → do **one Monitor pass** (below). If
  it finds work, the fix is this round-trip's job (`status = fixing` while pushing);
  finish the push+reply and return — don't loop.
- `status = merged` → do **Cleanup (step 6)** → status `done` → **the agent dies**.
- `status = done`/`failed` → nothing; should not have been spawned.

### Spec phase (analysis + plan; runs on opus/high effort, every task)
1. **Worktree + DESCRIPTIVE branch (off this task's BASE).** `git fetch origin`, then
   create a durable worktree whose branch is cut from this task's **base** — NOT main:
   - **0 or 2+ blockers** → base = the integration branch (`origin/hsb/<slug>-integration`).
   - **exactly 1 blocker** → base = that blocker's branch
     (`origin/hsb/<slug>-t<blockerId>-<blocker-slug>`) — a **stacked** branch.

   The branch name **must describe what the task does**, not just its id:
   ```
   hsb/<slug>-t<id>-<task-slug>
   ```
   where `<task-slug>` is a 2–5-word kebab-case summary of the task's actual work
   (from the title/goal), e.g. `hsb/reply-followups-t1-add-reply-correlation-matcher`.
   A generic `hsb/<slug>-t<id>` with no description is **not acceptable** — derive the
   slug from what you're building. Create it:
   `git worktree add .claude/worktrees/<branch> -b <branch> origin/<base>`.
   Record `branch`, `worktreePath`, and `baseBranch` (the resolved base) to
   `tasks/<id>.json` (so the Implement phase, PR creation, and rebases reuse the exact
   same base). Use the superpowers `using-git-worktrees` skill. NEVER work on `main`,
   and never branch a task off `main`.
2. **Spec → plan + complexity finding (autonomous).** Run the superpowers flow:
   `brainstorming` → `writing-plans`, doing real codebase analysis in context. At every
   checkpoint/decision, **pick the recommended option** and continue without blocking;
   append each non-trivial choice to the Decision Log `{decision, chosen, alternatives,
   why, howToRollback}`. **Determine this task's `complexity`** (`high|medium|low|
   trivial`, from the real touch set + shared surfaces uncovered) and write it to
   `tasks/<id>.json` — the orchestrator reads it to pick the Implement model, and the
   Implement phase reads it to pick the pre-push review depth. **`trivial`** is reserved
   for a change with **no logic/behavior change** — a typo, lint/formatting fix, or
   comment/doc-only edit; anything that alters behavior is at least `low`. End at
   `status = specified`. (Any sub-subagents you spawn for analysis also run opus/high.)

### Implement phase (TDD → PR; runs at modelPolicy[complexity])
3. **Implement (TDD).** Execute the plan via `executing-plans` /
   `subagent-driven-development` with `test-driven-development`. Honor all repo
   `CLAUDE.md` rules and invoke required project skills (migrations, etc.).
4. **Verify.** Run the repo's lint/format/tests gate (per `CLAUDE.md`). Use
   `verification-before-completion`. Do not open a PR on a red gate — fix first.
   Then run a **pre-push self-review scaled to `complexity`**, scoped to *this task's
   changes only* (the branch diff against its `baseBranch`, which is what `/code-review`
   reviews by default — never the whole repo):
   - **`trivial`** → **skip the review**; the green lint/format/tests gate is the whole
     bar. Push the quickfix. (Don't make a typo/lint/doc change bureaucratic.)
   - **`low` / `medium`** → run `/code-review low`.
   - **`high`** → run `/code-review high`.

   Treat findings as suggestions to judge on the merits (same JUDGE-BEFORE-YOU-ACT bar
   as PR comments): fix the real ones and re-run the gate; a clean/decline-only pass just
   proceeds. This self-review is *before the PR exists* and is distinct from the
   monitor/fix loop that answers reviewer comments on an already-open PR.
5. **PR (one per task, based on this task's `baseBranch`).** Commit on *this task's*
   branch, `git push -u origin <branch>` (this branch only, never main, never another
   task's branch), and open **this task's own** PR **targeting its `baseBranch`**:
   `gh pr create --base <baseBranch> --head <branch> --title "<title>"` — that base is
   the integration branch (0/2+ blockers) or the single blocker's branch (stacked).
   Resolve `<title>` via the **PR title convention** — match the cycle's latest PR
   using the `prTitlePattern` the top orchestrator passed in your brief (re-check the
   latest PR's current title if you can). Never `--base main` for a task. Do not append
   your changes onto a sibling task's branch/PR (unless this task qualifies for the
   trivial-fold exception in the rules, which the top orchestrator decides — a task
   agent always defaults to its own PR).
   **Open as `--draft` unless it is already safe to merge** (see the draft rule):
   any stacked PR, or any PR whose CI hasn't gone green yet, is created `--draft` so a
   human can't merge it out from under an unmerged base or in-flight work; un-draft
   (`gh pr ready`) only once settled and its base has merged.
   **Scale the PR body to `complexity` (see PR content requirements)** — a `trivial`/
   `low`/simple one-file change gets the short body, not the full template. Only `high` (and rich
   `medium`) tasks get the full `references/pr-template.md` treatment (Mermaid, full
   UAT, full decision table). Every PR still says what/why, how to test, and logs any
   non-trivial autonomous choice — but briefly, in plain language, and referencing
   sibling work by **PR number + one-line description**, never a bare task id. Write
   `prNumber/Url`, `branch`, `baseBranch`, `worktreePath`, `isDraft`, `decisionLog`,
   `status = open` to `tasks/<id>.json`; return the summary. The agent ends this tick
   here (it does NOT busy-wait for review).
6. **Cleanup (this agent's last act, once the human merges the task PR into
   integration).** On merge: `git worktree remove --force .claude/worktrees/<branch>`,
   delete the local branch, prune. Set `status = done` in `tasks/<id>.json`, sync the
   tracker (sub-issue → Done). **The agent dies** — the top orchestrator stops
   re-spawning it. (The integration branch + plan PR live until the human merges the
   plan PR into `main`.)

## Monitor pass (run BY the per-task agent, on its own settled PR)
The per-task agent — not the top orchestrator — does this when its task is **idle and
`status = open`** (PR up, no agent was in flight; see Idle-gating). It is never run
during a build or while another fix round-trip is in flight. It runs in the task's own
worktree and writes its own `tasks/<id>.json`. Use `gh` (verified at preflight). For
this task's PR `<n>`:

> ### NO SILENT FIXES (invariant — this is the bug this section exists to kill)
> **Every fix MUST be announced on the PR with a real `gh` post, and the post MUST
> be verified to exist before you consider the item handled.** A pushed commit is
> NOT a reply. The rule, with no exceptions:
> 1. Make the change in the worktree → 2. push the branch → 3. **post the reply/
> comment via `gh`** → 4. **capture the returned reply URL/id** → 5. only then record
> the item in `answeredComments` (store `{commentId, replyUrl}`).
> If step 3 returns no URL/id, it did NOT post — retry; do not advance. You may not
> return from this tick (or set `status` back to `open`) while any fix you pushed this
> tick lacks a recorded `replyUrl`. Before returning, assert: *every commentId acted
> on this tick has a `replyUrl`; every CI/conflict fix pushed this tick has a posted PR
> comment; and every **agreed-and-fixed** review item has its thread **resolved** and
> the reviewer **re-requested** (see SIGNAL RESOLUTION) so an automated reviewer sees
> it solved.* If not, do the missing posts/resolves now.

0. **Pull ALL signals first — every tick. "Not merged" is NOT "no change."** The
   classic bug is checking only `state,mergedAt`, seeing OPEN, and declaring "green,
   awaiting merge" — which is blind to requested changes, review threads, and CI. So
   begin EVERY tick with one comprehensive fetch and never short-circuit on merge
   state alone:
   ```
   gh pr view <n> --json state,mergedAt,isDraft,title,reviewDecision,reviewRequests,\
     latestReviews,statusCheckRollup,mergeable,mergeStateStatus
   ```
   plus the review/comment threads in step 2. You may NOT report a PR as "green /
   ready / awaiting merge" unless this tick you confirmed **all three**:
   `reviewDecision` is `APPROVED` (or null with no requested reviewers), CI
   (`statusCheckRollup`) is all green, and `mergeable` is clean. Otherwise report the
   true state (see below) and act on it.
1. **Merged?** If `mergedAt` set → do **Cleanup (step 6)**, set `status = done`
   in `tasks/<id>.json`, and **the agent dies**.
   - Also read `title`: if the human **renamed** this PR, include the new title as a
     `renamedTitle` field in your return summary so the **top orchestrator** updates
     `prTitlePattern` in `run.json` (you don't write `run.json`). It forward-propagates
     to the next PRs; never rename existing PRs to match.
2. **Reviews AND comments — read `reviewDecision` AND every comment thread (EVERY
   tick).** Interpret `reviewDecision` and report it accurately, don't collapse
   everything to "awaiting merge":
   - **`CHANGES_REQUESTED`** → actionable: address the feedback now (below).
   - **`REVIEW_REQUIRED` / requested reviewers pending** → *awaiting human review*,
     not awaiting merge. Report as "awaiting review from <reviewers>"; don't claim
     it's merge-ready.
   - **`APPROVED`** → eligible (still needs CI green + clean mergeable).
   **Pull ALL comment sources — a plain conversation comment counts as much as a
   formal review.** Do not check only reviews:
   - `gh api repos/{owner}/{repo}/issues/<n>/comments` — **general PR conversation
     comments** (the easy ones to miss; `gh pr view <n> --comments` also shows these),
   - `gh api repos/{owner}/{repo}/pulls/<n>/reviews` — each review's state + body
     (a review left as `COMMENTED` carries feedback too, not just `CHANGES_REQUESTED`),
   - `gh api repos/{owner}/{repo}/pulls/<n>/comments` — inline review-thread comments,
   - unresolved threads via GraphQL `reviewThreads { isResolved }` when available —
     an **unresolved thread is an unaddressed item even if you replied before**.

   **Every unanswered human comment is an actionable item** — a question to answer, a
   change to make, or a point to push back on — regardless of which source it came
   from or whether a formal review was submitted. (Skip only your own/bot comments.)

   > #### JUDGE BEFORE YOU ACT (reviews are suggestions, not commands)
   > Never blindly implement a comment/review. A reviewer can be wrong, working from
   > stale context, out of scope, or trading off something they can't see. For **each
   > unaddressed item**, FIRST evaluate it on the merits (use the superpowers
   > `receiving-code-review` skill — technical rigor, not performative agreement):
   > verify the claim against the actual code, weigh whether it is **correct, valuable,
   > reasonable, and necessary/in-scope**. Then pick a verdict:
   > - **Agree** → it's right and worth doing → implement it.
   > - **Alternative** → the concern is valid but the proposed fix isn't best →
   >   implement a better fix and explain why.
   > - **Decline** → wrong, unnecessary, or out of scope → make **NO** code change;
   >   reply with a respectful, evidence-backed reason and leave the thread open for
   >   the human (don't resolve a disagreement).
   > - **Clarify** → ambiguous or you're unsure of intent → ask a focused question in
   >   the reply; don't guess and don't change code yet.
   > Record each non-trivial agree/alternative/decline in the task `decisionLog` so the
   > human sees the reasoning. **Never edit-war:** if a human re-asserts after a
   > decline, re-judge honestly; if you still disagree, state it once more and leave it
   > to the human — never merge to end the disagreement.

   For each item, act on the verdict in its own worktree, then **post a real reply and
   verify it** (per the NO SILENT FIXES invariant):
   - If **Agree/Alternative** → make the change, push the branch, reply stating what
     you changed and why (+ the commit SHA).
   - If **Decline/Clarify** → push nothing; reply with the reasoned explanation or the
     question.
   - Inline/review-thread comment → reply **in-thread** so it threads under the
     reviewer's comment:
     `gh api repos/{owner}/{repo}/pulls/<n>/comments/<comment_id>/replies -f body='…'`
     (the response JSON's `html_url`/`id` is your proof — record it).
   - General PR/issue comment or a summary reply → `gh pr comment <n> --body '…'`
     (it prints the comment URL — record it).

   > #### SIGNAL RESOLUTION — a reply is NOT a resolution (this is why a bot reviewer
   > like **Macroscope** "doesn't understand the review was solved")
   > A reviewer — especially an automated one — tracks its findings by **thread
   > resolution state and review re-requests**, not by reading your prose reply. After
   > you **agree-and-fix** an item, you MUST actively signal it's resolved, or the bot
   > keeps showing it outstanding and `reviewDecision` stays `CHANGES_REQUESTED`:
   > 1. **Push the fix commit** (so the bot re-scans the new HEAD).
   > 2. **Resolve the review thread** for each fixed item — GraphQL
   >    `resolveReviewThread(threadId)` (get `threadId` from the
   >    `reviewThreads { id isResolved }` query in step 2). Only resolve threads you
   >    actually fixed; never resolve a **Declined/Clarify** thread (leave those for the
   >    human).
   > 3. **Re-request review** from the reviewer/bot once all its change-requests are
   >    handled, so it re-evaluates and can clear `CHANGES_REQUESTED`:
   >    `gh api repos/{owner}/{repo}/pulls/<n>/requested_reviewers -X POST -f
   >    reviewers[]='<reviewer-or-bot-login>'` (or `gh pr edit <n> --add-reviewer
   >    <login>`). For a bot that re-runs on push, the push + resolved threads is the
   >    trigger; re-request explicitly when it supports it.
   > 4. **Verify it took:** re-read `reviewDecision` + `reviewThreads.isResolved` next
   >    tick; if the bot still shows unresolved findings you believe are fixed, post a
   >    concise summary comment listing each finding → the commit/line that resolved it,
   >    and resolve the thread. Don't consider the PR review-clean until the reviewer's
   >    state reflects it.

   An item is "addressed" once it has been **judged, fixed-or-declined, replied to, AND
   its resolution signaled** (fixed → thread resolved + re-request; declined → reply
   only, thread left open). A posted reply alone is NOT "addressed" for an agreed fix.
   Record `{commentId, replyUrl, threadResolved}` in `answeredComments` ONLY after the
   reply is confirmed posted — never on push alone, so you never miss or double-post.
   Set `status = fixing` only while an *agreed* change is in flight, back to `open` once
   pushed, replied, **and resolution signaled** for every touched item.
3. **CI red?** Read `statusCheckRollup` from step 0; if any check is failing/pending,
   `gh pr checks <n>` for logs, fix in the worktree, push, then **`gh pr comment <n>
   --body '…'`** describing the failure + the fix + commit SHA, and record its URL. A
   CI fix without a posted comment violates NO SILENT FIXES.
4. **Merge conflict / behind base?** Rebase the branch onto **this task's `baseBranch`**
   (from `tasks/<id>.json`) — the integration branch (`origin/hsb/<slug>-integration`)
   for a 0/2+-blocker task, or the **blocker's branch** for a stacked task; for the
   **plan PR** it's `origin/main`. Never rebase onto `main` for a task. Resolve,
   force-push the branch (never main). **Because we flow instead of gating, a base
   advancing is expected, not exceptional:** when your blocker's branch gets new
   commits (or merges into integration and its branch is deleted — then re-base onto
   integration and update `baseBranch`), rebase onto the new base and keep going.
   **Post a `gh pr comment`** noting the rebase (and call out any behavior change), and
   record its URL.
5. Write `lastCheckedAt` and the updated `status` to `tasks/<id>.json`, run the NO
   SILENT FIXES assertion, then **return** your summary `{id, status, prNumber,
   merged?, renamedTitle?, prState, note}`. `prState` is the TRUE state observed this
   tick — one of `changes_requested | awaiting_review | ci_red | conflict |
   approved_green_awaiting_merge | merged` — so the orchestrator/console reports
   reality (e.g. "awaiting review", not "awaiting merge"). Only emit
   `approved_green_awaiting_merge` when reviewDecision=APPROVED **and** CI green **and**
   mergeable clean. **Never click merge** — leave that to the human.

### Plan-PR handling (top orchestrator; delegates the actual fixing)
- **Add a task's line to the plan PR body once, when its PR first opens**
  (`T-id — <title> — #<prNumber>`). After that, leave the body alone — don't tick,
  restyle, or re-edit it as CI runs or PRs merge. GitHub auto-renders the live status
  of every referenced PR, so churning the description each transition wastes time and
  adds noise. (Completion is signalled by the one-time un-draft + comment below, not
  by editing the body.)
- **When every task is `merged`/`done`:** mark the plan PR **ready for review**
  (`gh pr ready <planPrNumber>`), post a comment that the cycle is complete and it's
  ready to merge into `main`, move the **parent/epic tracker issue → In Review**.
- **Plan-PR fixes go DIRECTLY on the integration branch — no child PRs.** Once the
  cycle is in the plan-PR phase (tasks merged, only small review/CI comments left on
  the parent PR), feedback on the plan PR is committed straight to its own head,
  `hsb/<slug>-integration`. Do **not** open a child task PR (worktree→branch→PR→merge)
  for a small review fix — child PRs are for *task* work, not for tackling review
  comments on the finished cycle. (The integration branch is a feature branch, not
  `main`, so committing to it is allowed; the "one PR per task / branch off
  integration" rules apply to task work, not to plan-PR review fixes.)
- **How:** spawn a per-task-style agent **in the integration worktree** to run the
  same Monitor pass (JUDGE BEFORE YOU ACT + NO SILENT FIXES apply — judge each comment,
  fix the valid ones on the integration branch, reply to all), then push
  `hsb/<slug>-integration` directly. The top orchestrator never fixes inline.
  **Never merge it.**
  - Exception: if a plan-PR comment demands *substantial new feature work* (not a small
    fix), treat that as a new task — its own descriptive branch + child PR into
    integration — rather than a large direct commit. Default for review feedback is
    direct-to-integration.
- The run is "complete" only once a human merges the plan PR into `main`; then move
  the parent/epic issue → **Done**, remove the integration worktree/branch, and end
  the loop.

## Issue-tracker status sync (when the cycle is linked to Linear/Jira/etc.)
If the plan/tasks carry tracker issues, keep those issues' status in lockstep with
real delivery state, automatically and autonomously (no asking the user), on **every
status transition**. **Ownership follows the state split:** each **per-task agent**
syncs its OWN task's issue (it changes that task's status, so it owns the sync), and
the **top orchestrator** syncs the **parent/epic** issue (plan PR ready → In Review,
plan PR merged → Done). Because a sync fires whenever the owner writes a status, it
stays reliable across the re-spawn-each-tick model.

**Map to the project's REAL workflow states** (discovered at preflight), not these
literal names — pick the closest state each tracker actually has:

| Internal transition | Tracker action on the task's issue |
|---|---|
| → `implementing` (dispatched) | Move to **In Progress**; comment "🤖 started · branch `<branch>`" |
| → `open` (task PR opened) | Move to **In Review / Code Review**; comment the **PR link** |
| → `fixing` (review/CI feedback) | Keep In Review (use a "Changes Requested" state if one exists) |
| → `merged` (task PR merged into its base) | Move sub-issue to **Done/Merged**; comment "merged into `<baseBranch>` (cycle plan PR #<n>)" |
| → `failed` | Move to **Blocked**; comment the blocker + what's needed |
| plan PR `ready` (all tasks merged) | Move the parent/epic issue to **In Review** |
| plan PR **merged → `main`** | Move the parent/epic to **Done**; comment "shipped to main" |

Rules:
- **Idempotent.** Store `issue.lastSyncedStatus` per task; only transition/comment
  when it differs from the new state, so wakeups don't spam duplicate comments or
  re-fire transitions.
- **Map, don't invent.** Use the cached workflow-state list; if no clean match
  exists, choose the nearest and note the mapping once in the issue. Never create
  new workflow states.
- **Two-way, lightly.** If a human moved the issue (e.g. to Blocked) since last
  sync, respect it — comment rather than fight the human's manual change.
- **Best-effort, non-blocking.** A tracker write failing must not stall delivery:
  log it, leave `lastSyncedStatus` unchanged so the next wakeup retries, and keep
  driving the PR. (The PR, not the ticket, is the source of truth for the code.)

## PR title convention — always match the cycle's latest PR
Humans often rename a PR title to a house style; every new PR in the cycle must adopt
that style automatically, so the set stays visually consistent. **Before opening ANY
PR (task PR or plan PR), resolve the title from the cycle's most recent PR — never
just from a fixed template:**
1. From state, find the cycle's **most recently created PR** (max `createdAt` across
   task PRs + the plan PR) and re-fetch its **current** title:
   `gh pr view <n> --json title` (current, because you may have renamed it by hand).
2. If a cycle PR exists, **infer its title structure** — ticket/issue-key placement,
   leading prefix/scope (`feat:`, `[ABC-1234]`, `area:` …), separators,
   capitalization, emoji — and build this PR's title in the **same shape**, filled
   with this task's id/title. Save the exemplar + a short pattern note to state as
   `prTitlePattern` so the whole cycle stays consistent.
3. If **no cycle PR exists yet** (first PR of the run), match the repo's house style:
   `gh pr list --state all --limit 10 --json title` and follow the dominant pattern;
   if none is clear, default to `<source-key>: <Title>` (or just `<Title>`).
4. **Stay honest:** never fabricate a ticket key a task doesn't have. If the pattern
   carries one and this task has none, use the cycle's source key or omit that token.

The top orchestrator derives `prTitlePattern` and passes it in each task agent's brief; the agent
applies it when running `gh pr create --title`. During monitoring, if you rename a
cycle PR, capture the new title as the exemplar and update `prTitlePattern` so the
NEXT PRs follow the new pattern (see Monitor pass).

## PR content requirements — didactic, and scaled to complexity
Write PR bodies **for a human who has NOT been following the cycle**. Be didactic and
succinct: explain what the change is and how to test it in plain language, and don't
bury the reader in internal cross-references. **Right-size the body to the task's
`complexity`** — a one-file, low-complexity change must NOT get a huge multi-section
description.

**Two body sizes (pick by `complexity`):**
- **Simple body — `complexity: trivial`/`low` (and small `medium`):** a few sentences. Just:
  *What & why* (1–3 sentences), *How to test* (2–4 plain steps or a single line), and
  a *Decision log* **only if** a real choice was made (one line each). **No Mermaid,
  no acceptance-criteria checklist, no multi-section template.** A trivial one-file
  change gets a trivial PR body — matching the change's size is the point.
- **Full body — `complexity: high` (and rich `medium`):** the full
  `references/pr-template.md` — What & why, Mermaid diagram(s), full UAT, Decision
  log, Verification. Use this only when the change genuinely spans multiple files/
  services or has real design decisions worth a diagram.

**Rules that hold for every PR, both sizes:**
- **Be didactic about references.** Never point at an opaque internal id (`T2b`,
  "wave 2", "the matcher task") and expect the reader to decode it. When you must
  reference sibling work, give the **PR number + a one-line plain description** —
  e.g. "builds on #123 (adds the reply-correlation matcher)", not "depends on T2b".
  If a task id must appear, gloss it once: "T2 (PR #123 — the inbound pipeline)".
- **Decision Log — succinct, not a tech dump.** One line per non-trivial autonomous
  choice: *what was chosen*, *the alternative*, and *how to roll back*, in plain
  language. Leave out internal mechanics, file-level detail, and implementation
  narration — the human wants the decision and the off-ramp, not a design essay. If
  there were no real choices, omit the log entirely.
- **UAT teaches testing** — plain steps a non-author can follow (setup, action,
  expected result); for a simple change, one or two lines is enough.
- **Verification** — briefly, what tests/lint/gates ran and passed.

## Guardrails
- Push to `main`: forbidden, always. Merge a PR: forbidden **by default** — neither
  task PRs nor the plan PR — UNLESS the user has explicitly authorized merging for a
  named scope (task/wave/cycle); then merge only within that scope, only when green
  and not draft, and record the authorization in `run.json` + the Decision Log. Absent
  an explicit grant, leave every merge to the human.
- **Flow, never freeze.** Do not hold a task waiting for another's PR to merge —
  express the dependency by PR base (stack on the single blocker's branch, or target
  integration for 0/2+ blockers) and keep moving. The only synchronization point is a
  multi-blocker task's base = integration.
- **Keep unmergeable PRs as drafts.** A stacked PR, a PR with red/pending CI, or a PR
  whose agent still has work in flight stays `--draft` so a human can't merge it
  mid-flight; un-draft only when it's genuinely safe to merge.
- A task PR whose base is `main` is a defect: re-target it to its correct base
  (`gh pr edit <n> --base <baseBranch>` — integration or its blocker's branch) —
  never let task work merge straight to `main`.
- **Never conclude "no change / awaiting merge" from a merged-only check.** A
  `gh pr view --json state,mergedAt` that returns OPEN tells you nothing about
  reviews or CI. Every tick must pull `reviewDecision` + review threads + CI
  (`statusCheckRollup`) before reporting status — a PR with `CHANGES_REQUESTED`, an
  unresolved thread, or red CI is actionable, not "ready." The top orchestrator must
  not do this triage itself; it spawns the per-task agent, which runs the full
  Monitor pass (step 0 onward).
- If a task's gate cannot go green after honest attempts, set `status = failed`,
  leave the PR as draft with a clear blocker note, and surface it to the human —
  do not force a broken PR through.
- Keep the state directory the source of truth — `run.json` orchestrator-owned, each
  `tasks/<id>.json` owned by its task agent; locate + resume from it on every wakeup.
- Stay within scope of the task's brief; if the real diff diverges enough to create
  new cross-task conflicts, flag it (downstream waves may need re-planning).
