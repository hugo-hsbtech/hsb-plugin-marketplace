# Per-task agent playbook

**Audience: a per-task orchestrator agent** spawned by the cadence-executor top
orchestrator for ONE task (or for the plan PR, in monitor mode). Read this file
first, end to end, then act. Your brief from the orchestrator carries: the task's
context brief from the cycle plan (touch set + requirements + acceptance criteria),
your `runDir` and `tasks/<id>.json` path, the `integrationBranch`, the current
`prTitlePattern`, and the path to this file.

**Log your evidence as you go.** Append one JSON line to **your own**
`events/<id>.jsonl` at every milestone — `{ts, actor:"<id>", kind, detail, pr?, url?,
sha?, model?}` — the moment it happens: `task.spawn`, `task.complexity`, `join.built|
refreshed|retired`, `pr.opened`, `pr.ready`, `pr.redraft` (with the reason),
`decision.raised|reminded|answered|shipped_unresolved`, `review.received`,
`review.round`, `review.parked`, `comment.answered`, `ci.red|green`, `fix.pushed`,
`basesync`, `conflict`, `conflict.resolved`, `pr.retargeted`, `scope.checked`,
`scope.polluted`, `guard.blocked` (which guard, why), `automerge`, `merged`,
`gate.failed`, `escalation`, `task.failed` — **plus `state.changed` on every `status`
transition** (`from` → `to` + what caused it). Your task file holds only the current
value, so a transition you don't log is history that no longer exists: **if you wrote a
field, you owe an event.** It is append-only and yours alone — never
rewrite it, never touch another task's. The cycle report is rendered from these logs, so
an unlogged event is simply lost: **log the bad ones too** (re-drafts, failed gates,
blocked merges, parked reviewers). Schema and kinds:
`references/cycle-report.md`.

> ### EVERY TIMESTAMP COMES FROM `date -u` — AND THE KIND MUST MATCH THE DETAIL
> **You have no clock.** Get every `ts` you append, and every timestamped field you write
> to your task file (`specifiedAt`, `readyAt`, `lastCheckedAt`, `approvedAt`, …), from
> the shell:
>
> ```bash
> date -u +%Y-%m-%dT%H:%M:%SZ
> ```
>
> Never compose one from memory or inference, and never write local time with a `Z` on
> it — `Z` means UTC and nothing else. In production a whole run's agent-written
> timestamps were local time suffixed `Z` (three hours off GitHub's), which made every
> per-task duration in the report unusable: several tasks recorded a `readyAt` that
> *postdated their own `mergedAt`*. Where GitHub also stamps the moment (PR opened,
> merged, review submitted), take **GitHub's value** from `gh` — it is authoritative —
> and never mix the two sources inside one duration.
>
> **The `kind` vocabulary is closed: never invent one.** The report looks events up by
> `kind`, so an unrecognised kind is a *lost* event, not a richer log. Use a kind from
> the list above (or from `references/cycle-report.md`); if genuinely nothing fits, log
> an `incident` with a descriptive sub-`kind` — that is the one open-ended slot. And make
> the kind **match what you are about to write**: a `pr.redraft` whose detail says you
> only corrected the PR body, or a `gate.failed` whose detail says *"NOT a failure"*,
> are both the wrong kind — pick the one that describes what actually happened. (Both of
> those were logged in production, and both corrupted the report's health counts.)

You are **resumable and idempotent**: begin by reading your `tasks/<id>.json` and do
only the step your current `status` calls for, then write your own task file and
return. You are the **sole writer of your `tasks/<id>.json`** and the only one
running `gh` for your PR. You never write `run.json`, never touch another task's
branch/worktree/PR, and never push or commit to `main`. **On every status change,
run the Issue-tracker status sync** (bottom of this file) if the task is linked.

> ### YOUR TASK FILE IS WRITTEN BEFORE YOU RETURN — no exceptions
> `tasks/<id>.json` is the only durable record of your task, and you are its **sole
> writer**. The orchestrator cannot write it for you and will not go looking on GitHub
> to find out what you did. **Never return — successfully, early, or on a failure —
> without first writing your current `status` and everything you learned this
> invocation** (PR number/URL, branch, `baseBranch`, `complexity`, `isDraft` +
> `draftReason`, decisions, approvals, `filesChanged`, `scopeCheck`, `lastCheckedAt`),
> and appending the matching events to `events/<id>.jsonl`.
>
> In production an implement agent opened its PR, saw it merged, and returned without
> ever updating its task file: for the rest of the run the state said "still
> specifying, nothing implemented, nothing merged" while the PR had long since landed,
> and it had to be reconciled by hand against `gh`. **A step you completed but did not
> write down did not happen.** Write the file, then return.

## Resume dispatch (first thing you do)

- `status = pending` (or no file yet) → set `status = specifying`, run the **Spec
  phase** → write the `complexity` finding + plan. If complexity is `high`,
  `medium`, or `low`, end at `status = specified` and return. **Only if `trivial`,
  do NOT return — continue straight into the Implement phase in this same
  invocation** (the fused fast path) and end at `status = open`.
- `status = specifying` → spec was interrupted → resume the Spec phase (same fused
  rule applies).
- `status = specified` → set `status = implementing`, run the **Implement phase** →
  end at PR-opened, `status = open`. **Do NOT run a Monitor pass in this
  round-trip** — nothing to react to on a PR you just opened; the first monitor is
  a later idle tick.
- `status = implementing` → implement was interrupted → reconcile the worktree and
  continue the Implement phase (still ending at `open`, no monitor).
- `status = open` → settled, awaiting humans/CI → do **one Monitor pass** (below).
  If it finds work, the fix is this round-trip's job (`status = fixing` while
  pushing); finish the push+reply and return — don't loop. A monitor pass also
  re-runs the readiness checklist (un-drafting itself when it passes) and may end in
  `merged`→`done` if an intact human approval authorizes the merge.
- `status = merged` → do **Cleanup** → status `done` → **you die** (the
  orchestrator stops re-spawning you).
- `status = done`/`failed` → nothing; you should not have been spawned.

> **Why the fused fast path exists, and why it stops at `trivial`:** for a `trivial`
> task — a typo, a lint fix, a comment — spawning a second agent costs more than the
> change itself; you already hold the analysis, the worktree, and the branch, so you
> finish it here.
>
> It goes no further **because fusing forfeits the model tier**. You are running on the
> spec model (opus/high) and one invocation is one model, so anything you fuse is built
> on Opus regardless of what its `complexity` says. When `low` fused too, the cheapest
> tasks in a cycle ran on the most expensive model while `medium` ran on Sonnet — the
> cost curve inverted, and cycles ran nearly all-Opus. So `low`, `medium`, and `high`
> all return at `specified` and get their own implement agent at the right tier.

## Spec phase (analysis + plan; spawned on opus/high, every task)

1. **Worktree + DESCRIPTIVE branch (off this task's BASE).** `git fetch origin`,
   then create a durable worktree whose branch is cut from this task's **base** —
   NOT main:
   - **0 blockers** → base = the integration branch
     (`origin/cadence/<slug>-integration`).
   - **exactly 1 blocker** → base = that blocker's branch
     (`origin/cadence/<slug>-t<blockerId>-<blocker-slug>`) — a **stacked** branch.
   - **2+ blockers** → base = **your own join branch**, which you build now (below).
     You do **not** wait for the blockers to merge.

   > #### PUBLISH YOUR BRANCH IMMEDIATELY — before you write a line of code
   > As soon as the worktree exists, push the branch:
   > ```bash
   > git push -u origin <branch>      # empty at this point — identical to its base
   > ```
   > **Your branch existing on the remote is what makes every task that depends on you
   > dispatchable.** Until you push, they cannot start: their base does not exist, and
   > the orchestrator parks them as `waiting-for-blocker-branch`.
   >
   > This used to happen only at PR-creation time (Implement step 5), which meant a
   > dependent waited for your spec **and** your build **and** your gate **and** your
   > self-review **and** your PR body before it could begin. Chains of tasks therefore
   > ran end to end in sequence — the exact freeze this whole design exists to prevent,
   > wearing a different name. Pushing an empty branch costs nothing (it is identical to
   > its base, has no PR, and is invisible in review) and it unblocks your dependents a
   > tick after you *start* rather than after you *finish*.
   >
   > Record `branchPublishedAt` in `tasks/<id>.json` and log a `branch.published`
   > event. Then **keep pushing as you work** (Implement step 3) — never force-push.

   **Building the join branch (2+ blockers only).** You need a base that already
   contains every blocker's work; integration won't have it until they merge, and
   waiting for that is the freeze this exists to kill. So synthesize the base:
   ```bash
   git fetch origin
   git branch -f cadence/<slug>-t<id>-join origin/cadence/<slug>-integration
   git worktree add .claude/worktrees/cadence-<slug>-t<id>-join cadence/<slug>-t<id>-join
   # then, in that worktree, for EACH blocker branch:
   git merge --no-ff origin/cadence/<slug>-t<blockerId>-<blocker-slug>
   git push -u origin cadence/<slug>-t<id>-join
   ```
   - **Resolve cross-blocker conflicts here**, in the join — that's what the join is
     for. Log each resolution in `decisionLog` and summarize them in the PR body
     ("this PR sits on a join of #A and #B; they conflicted in `x.ts`, resolved by …").
     If two blockers conflict in a way you cannot resolve on the merits, that's an
     **open decision** (below), not a silent guess.
   - Record `joinBranch` **and `baseBranch = <integrationBranch>`** in
     `tasks/<id>.json`, plus `joinedShas` (the blocker head SHAs merged in) so you can
     tell later whether the join is stale. **The join is where you branch FROM; it is
     never your PR's base.**

   > **NOTHING EVER MERGES INTO A JOIN.** A join merges nowhere, so a PR merged *into*
   > one is stranded off integration — its source branch is auto-deleted, the work
   > disappears from the cycle, and the exit gate runs without it. Your PR therefore
   > targets **integration**, and if you ever find `baseRefName` pointing at a join,
   > re-target it now (`gh pr edit <n> --base <integrationBranch>`), log
   > `pr.retargeted`, and say so on the PR. Your join is **yours alone** — never build
   > on, or point at, another task's join.
   - A join branch is **infrastructure**: never open a PR for it, never review it, never
     merge anything into it, never share it, never target `main` with it.
   - **Refresh it** whenever a blocker's branch advances (Monitor step 4), and
     **retire it** once every blocker has merged into integration — just delete it,
     locally and remotely, and drop `joinBranch`. There is nothing to re-target: your PR
     was against integration all along, and its diff shrinks to your own work by itself
     as each blocker lands.

   The branch name **must describe what the task does**, not just its id:
   ```
   cadence/<slug>-t<id>-<task-slug>
   ```
   where `<task-slug>` is a 2–5-word kebab-case summary of the task's actual work
   (from the title/goal), e.g.
   `cadence/reply-followups-t1-add-reply-correlation-matcher`. A generic
   `cadence/<slug>-t<id>` with no description is **not acceptable**. Create it:
   `git worktree add .claude/worktrees/<branch> -b <branch> origin/<base>`.
   Record `branch`, `worktreePath`, and `baseBranch` (the resolved base) to
   `tasks/<id>.json` (so the Implement phase, PR creation, and base syncs reuse the
   exact same base). Use the superpowers `using-git-worktrees` skill. NEVER work on
   `main`, and never branch a task off `main`.

2. **Verify-and-extend the planner's brief — do NOT re-derive it.**

   > **Your blockers' handoffs outrank the plan's brief.** If your brief includes a
   > blocker's `handoff.contract`/`handoff.delivered`, that is the interface you build
   > against — it describes decisions a real agent made against real code, while the
   > plan's touch set was predicted before any of it existed. Where they disagree, take
   > the handoff and note the divergence in your own contract. If the handoff itself
   > looks wrong, say so in your return summary; don't silently route around it.

   The cycle
   plan's context brief (touch set + atomic requirement checklist + acceptance
   criteria) was already produced by opus/high codebase analysis. Re-running that
   analysis from scratch is the single biggest avoidable cost per task. Instead:
   - **Start from the brief.** Treat its touch set and R-id requirements as the
     working hypothesis.
   - **Verify it against the current code** (files drift between plan time and now).
     When a graphify graph is available (the orchestrator's preflight recorded
     `preflight.graphify = "ok"`, or `graphify-out/graph.json` exists), verify with
     graph queries first — `graphify query "…"`, `graphify path A B`,
     `graphify explain <Symbol>` — they answer imports/calls/dependents
     deterministically at zero LLM cost; read only the files the graph points at.
     Without graphify, spot-read the touch-set files directly.
   - **Escalate only on drift.** Spawn analysis sub-subagents (opus/high) ONLY for
     genuine unknowns: the brief is marked `analysis: low-confidence`, the code has
     materially diverged from the touch set, or a requirement has no anchor in the
     current code. Never fan out sub-subagents as a ritual re-analysis of what the
     brief already covers.
   - **Plan.** Use superpowers `writing-plans` to turn the verified brief into the
     implementation plan. Invoke `brainstorming` only when the brief leaves a real
     design decision open — not as a ceremony. At every checkpoint/decision, **pick
     the recommended option** and continue without blocking; append each
     non-trivial choice to the Decision Log
     `{decision, chosen, alternatives, why, howToRollback}`.
     **Exception:** a choice you cannot settle on the merits — because it needs
     information only a human has — is not a decision-log line. Raise it as an
     **open decision** (next section), keep your provisional default, and keep
     building. Do not stop, and do not ship it silently.
   - **If your task is a PREP BUNDLE** (its brief carries `Bundle class: <docs|config|
     schema>` and a list of scopes `T<id>.<n>`): treat it as ONE task with several
     scopes — one branch, one PR, one gate. Verify each scope's brief against the code
     as usual; its `complexity` is the **highest** of its scopes, never an average. A
     `schema` bundle is never `trivial` and **never skips the pre-push self-review**
     (minimum `/code-review low`), however small each migration looks. If a scope turns
     out not to belong — it changes behavior in a `docs` bundle, conflicts with another
     scope, or is simply bigger than the plan thought — **don't silently absorb it**:
     implement the rest, and report the misfit scope in your return summary so it can be
     re-planned as its own task. That is a finding, not a failure.
   - **Determine this task's `complexity`** (`high|medium|low|trivial`, from the
     verified touch set + shared surfaces) and write it to `tasks/<id>.json` — the
     orchestrator reads it to pick the Implement model, and the Implement phase
     reads it to pick the pre-push review depth. **`trivial`** is reserved for a
     change with **no logic/behavior change** — a typo, lint/formatting fix, or
     comment/doc-only edit; anything that alters behavior is at least `low`.

   - **Write your `handoff.contract`** into `tasks/<id>.json` before you return —
     this is what every task depending on you will build against, and the only thing
     that survives your turn. It states: the **exact** surfaces you will produce
     (names, signatures, routes, entities, migration names — not "an endpoint for X"),
     the design decisions you took and why, **anything that differs from the plan's
     brief**, and what a dependent must know to use it. Log `handoff.written`.

     > Your dependents' agents never share a context with you and never will. The plan
     > brief they also receive is a *prediction* written before any of this code
     > existed; your handoff is what is *true*. If you leave it vague, they guess — and
     > a guess about your interface is rework for both of you.

   Then: `high`/`medium`/`low` → write the plan, set `status = specified`, return
   (the orchestrator spawns the implement agent at that tier — `low` runs on
   sonnet/low, and that saving is the point). **Only `trivial`, and only when every
   blocker's PR is already open** → set `status = implementing` and continue below
   (fused path). A `trivial` task whose blockers aren't `open` yet stops at `specified`
   like everything else: fusing past a sequencing gate is how a dependent ships against
   code that does not exist.

## Open decisions (a question for the human must be ANSWERABLE, or it isn't asked)

> The bug this kills: a task hits a question it can't really answer, quietly picks a
> default, opens the PR, and the PR gets merged with the question never asked — or
> asked in a form nobody could act on. **If you need a human, leave them something
> they can answer in one reply, in the place they're already looking (the PR).**

**When is it an open decision?** Only when the answer needs information you do not
have and cannot derive from the code, the plan, or repo convention:
- product / UX intent (which behavior is actually wanted),
- a business or policy tradeoff (pricing, limits, retention, wording that carries
  commitment),
- ambiguous or contradictory acceptance criteria in the brief,
- an irreversible or data-affecting choice (destructive migration, backfill strategy,
  a public API/contract shape you'd have to break later),
- a security / compliance / privacy tradeoff,
- two blockers whose conflicting work can't be reconciled on technical merit.

Everything else stays **autonomous**: decide it, log it, move on. Raising too many
open decisions is its own failure — it turns an autonomous run into a questionnaire.
Aim for zero or one per task; three is a sign the brief was underspecified (say so).

**Blocking or not.** `blocking: true` when shipping the wrong answer would be wrong
behavior, hard to reverse after merge, or makes an acceptance criterion unverifiable.
Otherwise `blocking: false` — a reversible preference: default it, ask anyway, and
let it ride. A blocking decision **keeps the PR draft and disqualifies auto-merge**.

**Raising one (all five steps, or it doesn't count):**
1. **Keep building** with your provisional default so the branch stays complete and
   testable. Never park a task on an unanswered question.
2. **Post the question on the PR** as a comment, and capture its URL (same proof
   discipline as NO SILENT FIXES — no URL means it didn't post; retry):
   ```markdown
   ## 🔶 Open decision D2 — <one-line question>   <!-- blocking: yes -->

   **Why I can't decide this:** <what information is missing and why the code
   doesn't answer it — 1–2 sentences.>

   **Options**
   - **A — <label>** (currently implemented): <what it does> · <tradeoff>
   - **B — <label>**: <what it does> · <tradeoff>
   - **C — <label>**: <what it does> · <tradeoff>

   **In effect right now:** A. **Impact if we're wrong:** <blast radius, and how hard
   it is to change after merge.>

   **To answer:** reply on this comment with `D2: B` (or `D2: A`, `D2: C`, or just
   describe what you want). Until then this PR stays in draft and won't be merged.
   ```
   Ask for what you need, not for permission: never "shall I proceed?", never "do you
   want me to mark this ready?" — those are your calls, not theirs.
3. **Pin it in the PR body.** Keep an `## ⚠️ Open decisions` section at the very top
   of the body (this is the one part of the body you *do* keep current), one line per
   decision: `**D2** (blocking) — <question> → [answer here](<commentUrl>)`. Strike it
   through when resolved, noting the answer and the SHA that implemented it.
4. **Record it** in `tasks/<id>.json.pendingDecisions`:
   `{id: "D2", question, options, provisionalChoice, impact, blocking, commentUrl,
   status: "open", askedAt}`.
5. **Put it in a human's queue** — request review from the PR's human owner (or the
   cycle's requester) so it shows up where they triage: `gh pr edit <n> --add-reviewer
   <login>`. If the task is tracker-linked, comment the same question on the issue.

**Harvesting the answer** (every Monitor pass — see step 2b): scan new comments for a
reply to that thread or any comment naming the decision id. Any human answer wins over
your default, even a terse one (`D2: B`, "go with B", "keep A"). Then: implement it if
it differs from the default, reply with what you changed and the commit SHA, resolve
the thread, set `status: "resolved"` + `answeredBy`/`resolvedInSha`, update the PR body
line, and **re-run the readiness checklist** — clearing the last blocking decision is
usually what makes the PR ready.

**Reminders, not nagging.** If a blocking decision is still open after the run has gone
quiet twice (the orchestrator's backoff at its cap), post **one** short reminder
comment linking the original question, and set `remindedAt`. Never more than one
reminder per decision — after that it lives in the orchestrator's `attention` list,
which is printed every turn.

**Never let one evaporate.** You may not set `status = done`, un-draft, or auto-merge
with a blocking decision open. If a human merges the PR anyway with any decision
unresolved, post a follow-up comment on the merged PR stating which default therefore
shipped and how to change it, and report it in your return summary as
`unresolvedDecisions` so the orchestrator carries it into the final summary.

## Implement phase (TDD → PR; spawned at modelPolicy[complexity], or fused)

> ### FIRST: sync your base, then verify your producers are actually there
> You were only dispatched because **every blocker's PR is open** — their code is
> complete, gated and self-reviewed. So the code you depend on exists; your job is to
> pick it up and confirm it matches what you were told.
>
> 1. `git fetch origin` and **merge your base in** (`git merge origin/<baseBranch>`; for
>    a 2+-blocker task, refresh your join from integration + every blocker head first).
> 2. **Verify the surfaces you consume are present and match your blockers'
>    `handoff.delivered`** — the exact exports, endpoints, entities, columns or
>    migrations. Check the tree, not the brief.
>
> **If they're all there and match → build normally**, against the real signatures, not
> the ones the plan predicted.
>
> **If a surface is missing or differs from the handoff**, that is a **contract
> mismatch**, not something to work around. Do not stub it, invent it, or quietly adapt
> and hope. Record it (`implementBlockedOn: {taskId, surface, expected, found}`), log an
> `incident` (`kind: "contract.mismatch"`), raise it in your return summary so the
> orchestrator surfaces it, and — if you can still deliver something coherent without
> it — build that part and say exactly what you left out. Never poll or wait inside the
> tick (see NEVER WAIT INSIDE A TICK); return and let the next tick pick it up.

3. **Implement (TDD).** Execute the plan via `executing-plans` /
   `subagent-driven-development` with `test-driven-development` — for a `trivial`
   fused run, implement directly with TDD, no sub-orchestration.
   Honor all repo `CLAUDE.md` rules and invoke required project skills
   (migrations, etc.).

   > #### PUSH AS YOU GO — your commits are somebody else's base
   > `git push` after each meaningful commit, and **always** immediately after you
   > land a surface another task consumes (the exported function, the endpoint, the
   > entity, the migration your dependents were told to expect). Do not save it all
   > for the PR.
   >
   > A dependent stacks on *your branch*, so every commit you push is a commit it can
   > build against. Holding your work locally until the PR is open forces every task
   > below you to wait out your gate, your self-review and your PR body — for code you
   > finished writing an hour earlier. **The gate protects your PR, not your branch:**
   > a branch with no PR is not under review and nobody is reading it, so pushing
   > work-in-progress to it costs nothing and unblocks the cycle.
   >
   > Push plainly, and **never force-push** — a dependent may already be based on what
   > you pushed. If your local history needs fixing, fix it forward with another
   > commit.
4. **Verify.** Run the repo's lint/format/tests gate (per `CLAUDE.md`). Use
   `verification-before-completion`. Do not open a PR on a red gate — fix first.
   Then run a **pre-push self-review scaled to `complexity`**, scoped to *this
   task's changes only* (the branch diff against its `baseBranch`, which is what
   `/code-review` reviews by default — never the whole repo):
   - **`trivial`** → **skip the review**; the green lint/format/tests gate is the
     whole bar. Push the quickfix. (Don't make a typo/lint/doc change bureaucratic.)
   - **`low` / `medium`** → run `/code-review low`.
   - **`high`** → run `/code-review high`.

   Treat findings as suggestions to judge on the merits (same JUDGE-BEFORE-YOU-ACT
   bar as PR comments): fix the real ones and re-run the lint/format/tests gate; a
   clean/decline-only pass just proceeds.

   > #### SELF-REVIEW RUNS EXACTLY ONCE (one review, one fix pass, then the PR)
   > Do **NOT** run `/code-review` again on the fixed diff. A review at this depth
   > almost always finds *something* new, so re-reviewing after every fix loops
   > forever (review → fix → review → …) and the task never reaches `open` — the
   > cycle visibly stalls. After fixing, the only thing you re-run is the
   > lint/format/tests **gate**. Anything subtle the single pass missed is exactly
   > what the PR's human/bot reviewers are for.

   This self-review is *before the PR exists* and is distinct
   from the monitor/fix loop that answers reviewer comments on an already-open PR.
5. **PR (one per task, based on this task's `baseBranch`).** Commit on *this
   task's* branch, `git push -u origin <branch>` (this branch only, never main,
   never another task's branch), and open **this task's own** PR **targeting its
   `baseBranch`**:
   `gh pr create --base <baseBranch> --head <branch> --title "<title>"` — that base
   is the integration branch (no blockers) or the single blocker's branch (stacked).

   > **If you built a join branch (2+ blockers), your PR base is the INTEGRATION
   > BRANCH — never the join.** You *branched off* the join so your code compiles
   > against your blockers; you *target integration* so the work actually reaches the
   > cycle. A join has no PR of its own and is never a merge target, so a PR pointed at
   > one delivers its work into a dead end: in production four PRs merged into their
   > joins, vanished from integration, and needed three recovery PRs to get back. Set
   > `--base <integrationBranch>` and say in the body that the diff temporarily
   > includes unmerged blockers' files — it shrinks by itself as they land.

   Resolve `<title>` via the **PR title convention** (below). Never `--base main` for a
   task. Never append your changes onto a sibling task's branch/PR — one task, one
   branch, one PR, always.
   **Stamp your identity on the PR so nobody has to ask which task it is:**
   - the **first line of the body** is the identity header — in *both* body sizes:
     `**T2 · Wire the matcher into the inbound pipeline** — cycle \`<slug>\` · plan PR
     #<planPrNumber> · <tracker key, if any> · base \`<baseBranch>\` (stacked on
     #<basePr> — merge after it / join of #A + #B / the integration branch)`.
     Omit tokens that don't exist; never invent a ticket key.
   - **no labels, ever** — do not `--add-label`, do not `gh label create`, and do not
     ask for a label to be created. The repo's label set belongs to its humans; your
     identity header and the plan PR's cycle map are what make the PR findable.
   - return your `{id, title, prNumber}` so the orchestrator can add your row to the
     plan PR's cycle map.

   **Open as `--draft`, then immediately run the readiness checklist and un-draft
   yourself if it passes** (below). **Scale the PR body to `complexity` (see PR
   content requirements, below).**
   Write `prNumber/Url`, `branch`, `baseBranch`, `joinBranch?`, `worktreePath`,
   `isDraft`, `readyAt?`, `decisionLog`, `pendingDecisions`, `status = open` to
   `tasks/<id>.json`; return the summary.

   **Also write `handoff.delivered`** — what actually shipped, and it is the moment
   your dependents start moving, because their implement phase unlocks when your PR
   opens. State the **final** surfaces (exact names and signatures as merged into the
   branch, not as predicted), **any drift from your own `handoff.contract`** and why,
   and the gotchas you hit that a consumer would otherwise rediscover — the setup step
   that turned out to be required, the command that must be used and the one that
   silently fails, the ordering that matters. Log `handoff.written`.
   Cheap to write now; expensive for three other agents to re-derive later. You end this tick here (do NOT busy-wait
   for review).

   > #### READINESS IS YOURS TO DECIDE — never ask, never leave it hanging
   > **Draft means "a human would waste their time reading this," nothing else.**
   > Being stacked on an unmerged base is *not* a reason to stay draft: a stacked PR
   > merges into its blocker's *branch*, which is exactly where its code belongs, and
   > leaving it hidden is what strands a whole cycle in draft with nothing reviewable.
   > Say `Stacked on #N — merge after it` in the body instead.
   >
   > **Checklist — all five true → `gh pr ready <n>`:**
   > 1. no work of yours still in flight (you're about to return),
   > 2. local gate green (lint/format/tests) **and CI terminal-and-green on your
   >    CURRENT head SHA** — every check reported, none `QUEUED`/`IN_PROGRESS`/
   >    `PENDING`, none failed. **"No failing check" is NOT the test**: an empty or
   >    not-yet-created check set has no failing check either, and un-drafting on it
   >    invites review onto code nothing has verified (this shipped twice in
   >    production — once on "no CI checks report on this branch", once with two
   >    checks still `QUEUED`). Not terminal yet → stay draft with
   >    `draftReason: "ci_pending"`; the next tick un-drafts you when it goes green.
   >    Only a repo you have *confirmed* has no CI at all is exempt (record
   >    `ciExpected: false` and why),
   > 3. the pre-push self-review ran for this `complexity` (`trivial` → not required),
   > 4. **no unanswered blocking open decision**,
   > 5. the PR body is written to the standard (what & why, how to test, decisions).
   >
   > Then post one short comment ("ready for review — <one-line why>") and set
   > `isDraft = false`, `readyAt`. Any item false → stay draft and record which one in
   > `draftReason` so the orchestrator can report it honestly.
   >
   > **Never ask the user whether to mark a PR ready** — not here, not in a summary,
   > not on the next tick. It is a mechanical checklist, and it is yours. Equally,
   > never *report* a PR as ready while it is still a draft.
   >
   > Re-run this checklist on **every** monitor tick: a PR that went draft for red CI
   > un-drafts itself when CI goes green; a PR whose last blocking decision was just
   > answered un-drafts itself the same tick. And if a ready PR regresses (new
   > in-flight work, a new blocking decision, CI turns red), `gh pr ready --undo` and
   > post a one-line comment saying why.
6. **Cleanup (your last act, once the task PR is merged into its base — by the human,
   or by you under an intact human approval).**
   On merge: `git worktree remove --force .claude/worktrees/<branch>`, delete the
   local branch, prune. Set `status = done` in `tasks/<id>.json`, sync the tracker
   (sub-issue → Done). **You die** — the orchestrator stops re-spawning you. (The
   integration branch + plan PR live until the human merges the plan PR into
   `main`.)

## Monitor pass (on your own settled PR, status = open)

Runs only when the task is **idle and `status = open`** (PR up, no agent was in
flight). Never during a build or while another fix round-trip is in flight. Runs in
the task's own worktree; writes its own `tasks/<id>.json`. For this task's PR `<n>`:

> ### NEVER WAIT INSIDE A TICK — observe, act, return
> **A monitor pass is one observation and at most one round of action. It never
> sleeps, never polls, never loops "until CI finishes", and never waits for anything
> to change.** If CI is still running, that IS your answer: return immediately with
> `prState = ci_pending`. If a base sync is still settling, return. If you are waiting
> on a human, return. Waiting is the orchestrator's job — it owns the adaptive backoff
> and will re-spawn you on the next tick, when the thing you were waiting for has
> actually happened.
>
> In production a monitor agent did its base-sync correctly and then entered a
> wait-for-CI loop instead of returning a verdict. It re-notified four times, produced
> no action, burned roughly **80k subagent tokens per pass**, and had to be killed with
> `TaskStop`. It learned nothing that the next tick would not have learned for free.
> The loop is pure waste: your context is expensive and the orchestrator's sleep is
> free.
>
> Concretely, inside a tick: no `sleep`, no retry-until-green loop, no re-running the
> GraphQL call hoping for a different answer, no "wait a moment and check again". One
> fetch, judge, act on what is actionable now, write your task file, return.

> ### NO SILENT FIXES (invariant — this is the bug this section exists to kill)
> **Every fix MUST be announced on the PR with a real `gh` post, and the post MUST
> be verified to exist before you consider the item handled.** A pushed commit is
> NOT a reply. The rule, with no exceptions:
> 1. Make the change in the worktree → 2. push the branch → 3. **post the reply/
> comment via `gh`** → 4. **capture the returned reply URL/id** → 5. only then
> record the item in `answeredComments` (store `{commentId, replyUrl}`).
> If step 3 returns no URL/id, it did NOT post — retry; do not advance. You may not
> return from this tick (or set `status` back to `open`) while any fix you pushed
> this tick lacks a recorded `replyUrl`. Before returning, assert: *every commentId
> acted on this tick has a `replyUrl`; every CI/conflict fix pushed this tick has a
> posted PR comment; and every **agreed-and-fixed** review item has its thread
> **resolved** and the reviewer **re-requested** (see SIGNAL RESOLUTION) so an
> automated reviewer sees it solved.* If not, do the missing posts/resolves now.

0. **Pull ALL signals in ONE GraphQL call — every tick. "Not merged" is NOT "no
   change."** The classic bug is checking only `state,mergedAt`, seeing OPEN, and
   declaring "green, awaiting merge" — blind to requested changes, review threads,
   and CI. The second bug is paying 5–6 REST calls to learn it. One query returns
   everything:
   ```
   gh api graphql -F owner='{owner}' -F repo='{repo}' -F pr=<n> -f query='
   query($owner:String!,$repo:String!,$pr:Int!){
     repository(owner:$owner,name:$repo){ pullRequest(number:$pr){
       state mergedAt isDraft title reviewDecision headRefOid
       mergeable mergeStateStatus
       reviewRequests(first:20){nodes{requestedReviewer{
         ... on User{login} ... on Team{name}}}}
       reviews(last:30){nodes{author{login __typename} authorAssociation
         state body submittedAt commit{oid}}}
       comments(last:100){nodes{databaseId author{login __typename} body url createdAt}}
       reviewThreads(first:100){nodes{id isResolved
         comments(first:50){nodes{databaseId author{login} body url}}}}
       commits(last:1){nodes{commit{statusCheckRollup{state
         contexts(first:50){nodes{
           ... on CheckRun{name status conclusion detailsUrl}
           ... on StatusContext{context state targetUrl}}}}}}}
     }}
   }'
   ```
   That single response carries merge state, draft state, `reviewDecision`, every
   review (a `COMMENTED` review carries feedback too — and `author.__typename` +
   `authorAssociation` + `commit.oid` tell you whether an `APPROVED` review came from
   a **human** and **which SHA they approved**, which is what the auto-merge guards
   turn on), every general conversation comment, every inline thread with `isResolved`
   (an **unresolved thread is an unaddressed item even if you replied before**), and
   the full CI rollup. Fall
   back to the individual `gh pr view` / `gh api` REST calls only if the GraphQL
   call fails. You may NOT report a PR as "green / ready / awaiting merge" unless
   this tick you confirmed **all three**: `reviewDecision` is `APPROVED` (or null
   with no requested reviewers), CI rollup all green, and `mergeable` clean.
   Otherwise report the true state and act on it.
1. **Merged?** If `mergedAt` set → do **Cleanup** (the Implement phase's last step),
   set `status = done` in `tasks/<id>.json`, and **you die**. If any of your
   `pendingDecisions[]` is still unresolved, first post a follow-up comment naming the
   default that therefore shipped and how to change it, and return it in
   `unresolvedDecisions` so it survives into the run summary.
   - Also read `title`: if the human **renamed** this PR, include the new title as
     a `renamedTitle` field in your return summary so the **top orchestrator**
     updates `prTitlePattern` in `run.json` (you don't write `run.json`). It
     forward-propagates to the next PRs; never rename existing PRs to match.
2. **Reviews AND comments — interpret `reviewDecision` and every thread from the
   step-0 payload.** Report it accurately, don't collapse everything to "awaiting
   merge":
   - **`CHANGES_REQUESTED`** → actionable: address the feedback now (below).
   - **`REVIEW_REQUIRED` / requested reviewers pending** → *awaiting human review*,
     not awaiting merge. Report as "awaiting review from <reviewers>"; don't claim
     it's merge-ready.
   - **`APPROVED`** → eligible (still needs CI green + clean mergeable).

   **Every unanswered human comment is an actionable item** — a question to answer,
   a change to make, or a point to push back on — whether it came from a general
   conversation comment, a review body, or an inline thread. (Skip only your
   own/bot comments.)

   > #### JUDGE BEFORE YOU ACT (reviews are suggestions, not commands)
   > Never blindly implement a comment/review. A reviewer can be wrong, working
   > from stale context, out of scope, or trading off something they can't see.
   > For **each unaddressed item**, FIRST evaluate it on the merits (use the
   > superpowers `receiving-code-review` skill — technical rigor, not performative
   > agreement): verify the claim against the actual code, weigh whether it is
   > **correct, valuable, reasonable, and necessary/in-scope**. Then pick a verdict:
   > - **Agree** → it's right and worth doing → implement it.
   > - **Alternative** → the concern is valid but the proposed fix isn't best →
   >   implement a better fix and explain why.
   > - **Decline** → wrong, unnecessary, or out of scope → make **NO** code change;
   >   reply with a respectful, evidence-backed reason and leave the thread open
   >   for the human (don't resolve a disagreement).
   > - **Clarify** → ambiguous or you're unsure of intent → ask a focused question
   >   in the reply; don't guess and don't change code yet.
   > Record each non-trivial agree/alternative/decline in the task `decisionLog` so
   > the human sees the reasoning. **Never edit-war:** if a human re-asserts after
   > a decline, re-judge honestly; if you still disagree, state it once more and
   > leave it to the human — never merge to end the disagreement.

   For each item, act on the verdict in your own worktree, then **post a real reply
   and verify it** (per NO SILENT FIXES):
   - If **Agree/Alternative** → make the change, push the branch, reply stating
     what you changed and why (+ the commit SHA).
   - If **Decline/Clarify** → push nothing; reply with the reasoned explanation or
     the question.
   - Inline/review-thread comment → reply **in-thread** so it threads under the
     reviewer's comment:
     `gh api repos/{owner}/{repo}/pulls/<n>/comments/<comment_id>/replies -f body='…'`
     (the response JSON's `html_url`/`id` is your proof — record it).
   - General PR/issue comment or a summary reply → `gh pr comment <n> --body '…'`
     (it prints the comment URL — record it).

   > #### SIGNAL RESOLUTION — a reply is NOT a resolution (this is why a bot
   > reviewer often "doesn't understand the review was solved")
   > A reviewer — especially an automated one — tracks its findings by **thread
   > resolution state and review re-requests**, not by reading your prose reply.
   > After you **agree-and-fix** an item, you MUST actively signal it's resolved,
   > or the bot keeps showing it outstanding and `reviewDecision` stays
   > `CHANGES_REQUESTED`:
   > 1. **Push the fix commit** (so the bot re-scans the new HEAD).
   > 2. **Resolve the review thread** for each fixed item — GraphQL
   >    `resolveReviewThread(threadId)` (you already have `threadId` from the
   >    step-0 payload). Only resolve threads you actually fixed; never resolve a
   >    **Declined/Clarify** thread (leave those for the human).
   > 3. **Re-request review** from the reviewer/bot once all its change-requests
   >    are handled, so it re-evaluates and can clear `CHANGES_REQUESTED`:
   >    `gh api repos/{owner}/{repo}/pulls/<n>/requested_reviewers -X POST -f
   >    reviewers[]='<reviewer-or-bot-login>'` (or `gh pr edit <n> --add-reviewer
   >    <login>`). For a bot that re-runs on push, the push + resolved threads is
   >    the trigger; re-request explicitly when it supports it.
   > 4. **Verify it took:** re-read `reviewDecision` + `reviewThreads.isResolved`
   >    next tick; if the bot still shows unresolved findings you believe are
   >    fixed, post a concise summary comment listing each finding → the
   >    commit/line that resolved it, and resolve the thread. Don't consider the PR
   >    review-clean until the reviewer's state reflects it.

   > #### REVIEW CONVERGENCE BOUND (max 3 fix rounds per reviewer, then park)
   > An automated reviewer re-reviews every push and will usually emit *something*
   > new each time, so fix → re-request → new review → fix … can cycle forever —
   > the PR churns, the run never quiets, and the backoff never engages. Bound it
   > **per reviewer**:
   > 1. Track rounds in `tasks/<id>.json` → `reviewerFixRounds[<reviewer-login>]`.
   >    A "round" = one monitor tick in which you pushed agreed fixes for that
   >    reviewer's findings and re-requested (or re-triggered) them.
   > 2. **Rounds 1–3:** normal flow — judge → fix/decline → reply → resolve →
   >    re-request.
   > 3. **After round 3, if the same reviewer still returns new findings: STOP
   >    re-requesting that reviewer.** Still judge each outstanding item and reply
   >    (JUDGE BEFORE YOU ACT applies as always — fix anything genuinely broken),
   >    but post ONE summary comment instead of another re-request: the rounds
   >    completed, what was fixed (commit SHAs), what was declined and why, and
   >    that this reviewer's remaining findings now await the human's judgment.
   > 4. Report the honest `prState` (e.g. `changes_requested`) with a note like
   >    `"review loop parked after 3 rounds with <reviewer> — human decision
   >    needed"`. With no re-request there is no new review, so the PR goes quiet
   >    and the orchestrator's backoff takes over instead of burning ticks on churn.
   >
   > The bound is per reviewer, not per PR: a comment from a **human** or a
   > **different** reviewer is handled normally regardless of the counter. And the
   > cap stops *re-request churn only* — red CI, merge conflicts, and genuinely
   > broken code are always fixed, whatever the round count.

   An item is "addressed" once it has been **judged, fixed-or-declined, replied to,
   AND its resolution signaled** (fixed → thread resolved + re-request; declined →
   reply only, thread left open). A posted reply alone is NOT "addressed" for an
   agreed fix. Record `{commentId, replyUrl, threadResolved}` in `answeredComments`
   ONLY after the reply is confirmed posted — never on push alone, so you never
   miss or double-post. Set `status = fixing` only while an *agreed* change is in
   flight, back to `open` once pushed, replied, **and resolution signaled** for
   every touched item.
2b. **Harvest answers to your open decisions, and keep them visible.** For each
   `pendingDecisions[]` entry with `status: "open"`, look in this tick's payload for a
   human answer — a reply in that thread, or any comment naming the id (`D2: B`, "go
   with B on D2", "keep the default"). A human answer always wins over your default.
   - **Answered** → if it differs from what's implemented, make the change in your
     worktree and push; then reply with what you changed + the commit SHA, resolve the
     thread, set `status: "resolved"`, `answeredBy`, `answeredAt`, `resolvedInSha`,
     and strike the line through in the PR body's **⚠️ Open decisions** section. If
     the answer confirms the default, still reply, resolve, and mark it resolved — an
     unanswered-looking decision is as bad as an unasked one.
   - **Ambiguous answer** → reply once asking the narrow follow-up; keep it open.
   - **Still open** → leave it; post the single reminder if the run has been quiet
     twice and `remindedAt` isn't set. Never re-post the question.
   - After any resolution, **re-run the readiness checklist** (step 5) — clearing the
     last blocking decision is usually what makes the PR ready.
   - A new open decision can also arise *here* (a reviewer raises a product question
     you can't settle): raise it with the full five-step protocol, which means the PR
     goes back to draft if it's blocking.
2c. **Update the approval ledger** (this is what makes auto-merge safe). From the
   `reviews` in this tick's payload, maintain `approvals[]` in `tasks/<id>.json`:
   `{login, isHuman: author.__typename == "User", association, state, sha:
   commit.oid, at: submittedAt}`. Keep the **first** SHA at which each reviewer
   approved (`approvedSha`) and clear a reviewer's approval when they later submit
   `CHANGES_REQUESTED` or their review is dismissed. This ledger is append-only
   bookkeeping — it never decides anything by itself; step 6 checks the guards.
3. **CI red?** Read the `statusCheckRollup` from step 0; if any check is
   failing/pending, `gh pr checks <n>` for logs, fix in the worktree, push, then
   **`gh pr comment <n> --body '…'`** describing the failure + the fix + commit
   SHA, and record its URL. A CI fix without a posted comment violates NO SILENT
   FIXES.
4. **Base moved / behind / conflicted? Sync NOW — this is the highest-priority work
   in a monitor tick.** The human merges PRs while the cycle runs, so a base advancing
   under you is routine, not exceptional. A conflicted PR is unreviewable *and* shows
   the wrong diff, so it must never sit waiting to be noticed.

   **4a. Reconcile your base first.** Compare the PR's real `baseRefName` (from the
   step-0 payload) with your `baseBranch`. **GitHub silently re-targets a PR when its
   base branch is deleted** — merge a stacked parent with "delete branch" on and your
   PR is suddenly based on the grandparent. If they differ, update `baseBranch` (and
   `joinBranch` if it retired) *before* syncing, and say so in the sync comment.
   - **Stacked (1 blocker)** → base = `origin/<blocker branch>`; if that blocker merged
     and its branch is gone, base = the integration branch
     (`gh pr edit <n> --base <integrationBranch>`).
   - **Join base (2+ blockers)** → refresh the join whenever a blocker head moved
     (compare against `joinedShas`): in the join worktree, merge current integration +
     each blocker's new head, push the join, then sync your branch onto it. Once
     **every** blocker has merged into integration, **retire the join** — re-target the
     PR to integration, set `baseBranch`, drop `joinBranch`, delete the join branch.
   - **No blockers** → base = the integration branch. For the **plan PR** it's
     `origin/main`. Never sync a task onto `main`.

   **4b. Merge the base in — NEVER rebase, NEVER force-push an open PR.** A human may
   be mid-review: a force-push unanchors their inline comments and can dismiss their
   approval. Prefer GitHub's own no-op-for-you path, then fall back to local:
   ```bash
   gh api repos/{owner}/{repo}/pulls/<n>/update-branch -X PUT   # merges base into head
   # if that fails or reports conflicts, do it in your worktree:
   git fetch origin && git merge origin/<baseBranch>            # resolve, then:
   git push                                                     # plain push, no --force
   ```

   **4c. Resolve a squash-merge collision in the BASE's favor.** The classic conflict
   after a human merges: the blocker's work landed on your base as **one squashed
   commit**, while your branch still carries the blocker's **original** commits — the
   same change exists twice, so git conflicts, and the PR starts showing the blocker's
   files as if they were yours. For every hunk that is your copy of a change **already
   merged into the base**, take the **base's** version — the blocker's work is
   authoritative there and your copy is a stale duplicate. Keep your own work untouched.
   If you cannot tell whose change a hunk is, check whether the base already contains it
   (`git log origin/<base> --oneline -S'<distinctive line>'`) before deciding.

   **4d. Scope check — the diff must be your task's work and nothing else.**
   ```bash
   git diff origin/<baseBranch>...HEAD --name-only     # three dots: vs the merge base
   ```
   Every path must be one this task legitimately touches (its brief's touch set plus
   anything the spec phase justified) — **or a file belonging to an unmerged blocker you
   are built on**, which is expected and must be **declared, not hidden**: your branch
   sits on top of that blocker's work until it lands in integration. Declare it **as a
   condition, not a count** — *"Built on #35; until that merges, its files appear in this
   diff too."* That sentence stays true until it stops applying, and it says so itself.
   **Never write a file count into the body** — GitHub renders it live and the next
   commit makes yours a lie (see DURABLE, NOT VOLATILE). The count belongs in
   `tasks/<id>.json.filesChanged` and the `scope.checked` event, where current-state
   lives. Anything in neither category means the sync went wrong — almost always 4c
   resolved the wrong way. **Fix it and re-check; do not push a polluted diff
   and do not invite review onto one.** Record `filesChanged` + the verdict in
   `tasks/<id>.json` — state, not prose. If the diff legitimately reaches beyond the
   obvious, explain **why** in the body's scope line, qualitatively; never a number. An
   unexplained sprawling diff for a small task is a defect, not a big task.

   **4e. Announce it.** `gh pr comment <n>` describing what moved, what you merged in,
   how any collision was resolved, and the resulting file count — record the URL (NO
   SILENT FIXES). Log `basesync`/`conflict`/`conflict.resolved`/`scope.checked` events. Then re-run the readiness
   checklist: a PR that went draft for a conflict un-drafts itself once clean.

   > **Never report "waiting for #N to merge" as your state.** If your base advancing
   > is what you need, sync now — that's this step. The only things you legitimately
   > wait on are a human (review, decision) and CI.
5. **Re-run the readiness checklist** (Implement phase, step 5's block). Un-draft with
   `gh pr ready <n>` the moment all five items pass — most often this is the tick
   where CI went green, the last blocking decision got answered, or the fix round
   finished. If a ready PR regressed, `gh pr ready --undo` + one-line comment. Update
   `isDraft` / `readyAt` / `draftReason`. **This is never a question you ask.**
6. **Approval-authorized auto-merge (task PRs only; never the plan PR).** An approving
   review from a **human** on your PR is that human's authorization to merge it — so
   merge it, instead of parking finished work behind a button. Merge **only if all of
   these hold in THIS tick's payload** (skip entirely when
   `run.json.approvalMergePolicy = "off"`, or when your base is `main`):
   1. `approvals[]` contains a **human** approval (`__typename == "User"`) — a bot
      approval alone never authorizes a merge;
   2. `reviewDecision == APPROVED`, with no later `CHANGES_REQUESTED` or dismissal;
   3. **the approval still describes the code** — see the staleness test below;
   4. **no `pendingDecisions[]` entry is unresolved** — blocking or not;
   5. every human comment/thread has a recorded `replyUrl`, and every agreed-and-fixed
      thread `isResolved`;
   6. CI rollup fully green — nothing failing, no required check pending;
   7. `mergeable == MERGEABLE` and `mergeStateStatus` not in
      `DIRTY / BLOCKED / BEHIND / UNSTABLE`;
   8. **your base is landable** — it is the integration branch. If your base is another
      task's branch whose PR is **still open**, or your **join branch**, do NOT merge:
      merging would inject your commits into a parent PR nobody reviewed them in (or
      into infrastructure). Keep the approval in `approvals[]`, report
      `approved_awaiting_base`, and merge on the tick after your base lands (or after
      the join retires and you re-target to integration). **This is not a freeze** —
      the PR is ready, your dependents already build on your branch, and only the
      button waits.

   > **Staleness test (guard 3) — "no considerable change since approval."** Compare
   > `headRefOid` with the `approvedSha` from your ledger. The approval survives only
   > when the head is:
   > - **identical** to `approvedSha`; or
   > - a **base-sync merge** — the only new commits merge the updated base (plus
   >   conflict resolutions taking the base's already-merged version); your own patch set against its
   >   merge-base is unchanged (compare `git patch-id` over
   >   `git diff <mergeBase>..<approvedSha>` and `git diff <mergeBase>..HEAD`); or
   > - **`trivial`-class only** — the post-approval diff touches nothing but comments,
   >   docs, or formatting, with no behavior change.
   >
   > Any other change — logic, tests, config, dependencies, migrations, CI — makes the
   > approval **stale**. **When in doubt, stale.** Then: do **not** merge; post a
   > comment saying the approval predates `<what changed>` and re-request the
   > approver; wait for a fresh one.

   **If every guard passes:**
   1. If still draft → `gh pr ready <n>` (a draft PR can't be merged).
   2. **Post the authorization record first, and capture its URL** — who approved, at
      which SHA, what changed since (nothing / base-sync / trivial), and each guard that
      passed. **NO SILENT MERGES**: a merge nobody can audit from the PR itself is the
      same defect as a silent fix.
   3. Merge into your base with a method the repo allows (`gh repo view --json
      squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed`; prefer squash):
      `gh pr merge <n> --squash`. **Do not pass `--delete-branch`** while another
      task's `baseBranch` or join branch may point at your branch — the dependent
      re-targets on its next tick, and Cleanup removes the branch anyway.
   4. Record `autoMerge {authorizedBy, approvedSha, mergedSha, headDelta, guards,
      mergedAt, commentUrl}` and append it to `decisionLog`; set `status = merged`.
   5. Continue straight into **Cleanup** in this same invocation, then you die.

   **If any guard fails, don't merge and don't go quiet about it:** reply/comment with
   the specific guard that failed (e.g. "approval from @x is at `abc123`; head is
   `def456` with logic changes — re-requesting review"), and report the true state.
7. Write `lastCheckedAt` and the updated `status` to `tasks/<id>.json`, run the NO
   SILENT FIXES assertion, then **return** your summary `{id, status, prNumber,
   merged?, autoMerged?, renamedTitle?, prState, openDecisions, note}`. `prState` is
   the TRUE state observed this tick — one of `changes_requested | awaiting_review |
   awaiting_human_decision | ci_pending | ci_red | conflict | approved_awaiting_base |
   approved_green_awaiting_merge |
   auto_merged | merged` — so the orchestrator/console reports reality (e.g. "awaiting
   review", not "awaiting merge"). Only emit `approved_green_awaiting_merge` when
   reviewDecision=APPROVED **and** CI green **and** mergeable clean **and** an
   auto-merge guard held it back (say which in `note`). `openDecisions` carries every
   unresolved decision `{id, question, commentUrl, blocking}` so the orchestrator can
   put it in the run's `attention` list. **Never merge the plan PR** — that gate is
   the human's, always.

## PR title convention — always match the cycle's latest PR

Humans often rename a PR title to a house style; every new PR in the cycle must
adopt that style automatically, so the set stays visually consistent. **Before
opening ANY PR, resolve the title from the cycle's most recent PR — never just from
a fixed template:**
1. Your brief carries the orchestrator's current `prTitlePattern` (exemplar +
   pattern note). Re-check the exemplar PR's *current* title if you can
   (`gh pr view <n> --json title`) — a human may have renamed it.
2. **Infer the title structure** — ticket/issue-key placement, leading prefix/scope
   (`feat:`, `[ABC-1234]`, `area:` …), separators, capitalization, emoji — and
   build this PR's title in the **same shape**, filled with this task's id/title.
3. If **no cycle PR exists yet** (first PR of the run), match the repo's house
   style: `gh pr list --state all --limit 10 --json title` and follow the dominant
   pattern; if none is clear, default to `<source-key>: <Title>` (or just
   `<Title>`).
4. **Stay honest:** never fabricate a ticket key a task doesn't have. If the
   pattern carries one and this task has none, use the cycle's source key or omit
   that token.

If during monitoring you observe the human renamed a cycle PR, report the new title
as `renamedTitle` in your return summary — the orchestrator updates
`prTitlePattern` so the NEXT PRs follow the new pattern.

## PR content requirements — didactic, and scaled to complexity

Write PR bodies **for a human who has NOT been following the cycle**. Be didactic
and succinct: explain what the change is and how to test it in plain language, and
don't bury the reader in internal cross-references. **Right-size the body to the
task's `complexity`** — a one-file, low-complexity change must NOT get a huge
multi-section description.

**Two body sizes (pick by `complexity`):**
- **Simple body — `complexity: trivial`/`low` (and small `medium`):** a few
  sentences. Just: *What & why* (1–3 sentences), *How to test* (2–4 plain steps or
  a single line), and a *Decision log* **only if** a real choice was made (one line
  each). **No Mermaid, no acceptance-criteria checklist, no multi-section
  template.** A trivial one-file change gets a trivial PR body — matching the
  change's size is the point.
- **Full body — `complexity: high` (and rich `medium`):** the full
  `references/pr-template.md` — What & why, Mermaid diagram(s), full UAT, Decision
  log, Verification. Use this only when the change genuinely spans multiple
  files/services or has real design decisions worth a diagram.

**Rules that hold for every PR, both sizes:**
- **DURABLE, NOT VOLATILE — the body is a description, not a dashboard.** It is written
  once and read for days, so it must still be true next week. **Never state anything
  GitHub already renders live and the next commit can falsify:** file or line counts,
  test pass counts, coverage, timings, "CI green", "all tests passing", mergeability,
  draft/review state, "N of M tasks merged". Write intent instead — what changed and
  why, scope *in words*, the commands to verify (not their results), decisions and how
  to roll them back, risks. **Events with their reasons are history, not status**, and
  are welcome ("#37 merged into the base and renamed the tables; the conflict was
  resolved keeping both intents"). Test each sentence: *would one more commit make this
  wrong?* If yes, it belongs in a **comment** — timestamped, reading as an event — not
  in the body. The sole exception is the **⚠️ Open decisions** section, which is a live
  checklist by design.
- **The identity header is the first line, in both sizes.** `**<T-id> · <what this task
  does, in plain words>** — cycle <slug> · plan PR #<n> · <tracker key> · base <...>`.
  A reader landing cold on this PR must learn, without asking anyone: what it does,
  which task it is, which cycle it belongs to, and what it's based on.
- **Open decisions go immediately after the header, in both sizes.** If any exist, an
  `## ⚠️ Open decisions` section — one line each: `**D<n>** (blocking) — <question> →
  [answer here](<commentUrl>)`, struck through once resolved with the answer and the
  SHA. This is the **only** part of the body you keep updating; everything else is
  written once. No open decisions → omit the section entirely.
- **A prep bundle gets one labeled section per scope.** `### Scope 2 of 4 — <what>
  (T0.2)`, each with its own what/why, how-to-test, and roll-back line — so a reviewer
  can read, test, and judge each scope independently inside one PR. State once, near the
  top: *"These N scopes revert as a set — say so before merge if any needs to be
  independently revertible."* Never merge the scopes into one undifferentiated
  narrative; that's the thing that makes a multi-scope PR unreviewable.
- **State the stacking in words, not in the draft flag.** The header's `base` token
  says it in plain language: `Stacked on #123 (adds the matcher) — merge after it`, or
  `Sits on a join of #123 + #124`. A reviewer must be able to tell what they're looking
  at without decoding branch names.
- **Be didactic about references.** Never point at an opaque internal id (`T2b`,
  "wave 2", "the matcher task") and expect the reader to decode it. When you must
  reference sibling work, give the **PR number + a one-line plain description** —
  e.g. "builds on #123 (adds the reply-correlation matcher)", not "depends on
  T2b". If a task id must appear, gloss it once: "T2 (PR #123 — the inbound
  pipeline)".
- **Decision Log — succinct, not a tech dump.** One line per non-trivial autonomous
  choice: *what was chosen*, *the alternative*, and *how to roll back*, in plain
  language. Leave out internal mechanics, file-level detail, and implementation
  narration. If there were no real choices, omit the log entirely.
- **UAT teaches testing** — plain steps a non-author can follow (setup, action,
  expected result); for a simple change, one or two lines is enough.
- **Verification** — briefly, what tests/lint/gates ran and passed.

## Issue-tracker status sync (when this task is linked to Linear/Jira/etc.)

You sync YOUR OWN task's issue on **every status transition**, automatically and
autonomously (the top orchestrator syncs only the parent/epic issue). **Map to the
project's REAL workflow states** (discovered at preflight, cached in `run.json`'s
`issueTracker`), not these literal names — pick the closest state the tracker
actually has:

| Internal transition | Tracker action on the task's issue |
|---|---|
| → `implementing` (dispatched) | Move to **In Progress**; comment "🤖 started · branch `<branch>`" |
| → `open` (task PR opened) | Move to **In Review / Code Review**; comment the **PR link** |
| → `fixing` (review/CI feedback) | Keep In Review (use a "Changes Requested" state if one exists) |
| **open decision raised** (blocking) | Use a **Blocked / Needs Info** state if one exists (else keep In Review); comment the **question + the PR link where it's answered** |
| **open decision resolved** | Back to **In Review**; comment the answer and what changed |
| → `merged` (task PR merged into its base) | Move sub-issue to **Done/Merged**; comment "merged into `<baseBranch>` (cycle plan PR #<n>)" — say **who authorized it** when it was an approval-authorized auto-merge |
| → `failed` | Move to **Blocked**; comment the blocker + what's needed |

Rules:
- **Idempotent.** Store `issue.lastSyncedStatus` in your task file; only
  transition/comment when it differs from the new state, so wakeups don't spam
  duplicate comments or re-fire transitions.
- **Map, don't invent.** Use the cached workflow-state list; if no clean match
  exists, choose the nearest and note the mapping once in the issue. Never create
  new workflow states.
- **Two-way, lightly.** If a human moved the issue (e.g. to Blocked) since last
  sync, respect it — comment rather than fight the human's manual change.
- **Best-effort, non-blocking.** A tracker write failing must not stall delivery:
  log it, leave `lastSyncedStatus` unchanged so the next wakeup retries, and keep
  driving the PR. (The PR, not the ticket, is the source of truth for the code.)
