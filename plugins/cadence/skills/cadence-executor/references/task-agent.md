# Per-task agent playbook

**Audience: a per-task orchestrator agent** spawned by the cadence-executor top
orchestrator for ONE task (or for the plan PR, in monitor mode). Read this file
first, end to end, then act. Your brief from the orchestrator carries: the task's
context brief from the cycle plan (touch set + requirements + acceptance criteria),
your `runDir` and `tasks/<id>.json` path, the `integrationBranch`, the current
`prTitlePattern`, and the path to this file.

You are **resumable and idempotent**: begin by reading your `tasks/<id>.json` and do
only the step your current `status` calls for, then write your own task file and
return. You are the **sole writer of your `tasks/<id>.json`** and the only one
running `gh` for your PR. You never write `run.json`, never touch another task's
branch/worktree/PR, and never push or commit to `main`. **On every status change,
run the Issue-tracker status sync** (bottom of this file) if the task is linked.

## Resume dispatch (first thing you do)

- `status = pending` (or no file yet) → set `status = specifying`, run the **Spec
  phase** → write the `complexity` finding + plan. If complexity is `high` or
  `medium`, end at `status = specified` and return. If **`trivial` or `low`, do NOT
  return — continue straight into the Implement phase in this same invocation**
  (the fused fast path) and end at `status = open`.
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
  pushing); finish the push+reply and return — don't loop.
- `status = merged` → do **Cleanup** → status `done` → **you die** (the
  orchestrator stops re-spawning you).
- `status = done`/`failed` → nothing; you should not have been spawned.

> **Why the fused fast path exists:** for a `trivial`/`low` task, spawning a second
> agent to implement costs more than the implementation itself — the fresh agent
> re-reads everything you already have in context. You already hold the analysis,
> the worktree, and the branch; finishing the small change here is both cheaper and
> better (it runs on the spec model). `medium`/`high` tasks always return at
> `specified` so the orchestrator can spawn the implement agent on the right model
> tier.

## Spec phase (analysis + plan; spawned on opus/high, every task)

1. **Worktree + DESCRIPTIVE branch (off this task's BASE).** `git fetch origin`,
   then create a durable worktree whose branch is cut from this task's **base** —
   NOT main:
   - **0 or 2+ blockers** → base = the integration branch
     (`origin/cadence/<slug>-integration`).
   - **exactly 1 blocker** → base = that blocker's branch
     (`origin/cadence/<slug>-t<blockerId>-<blocker-slug>`) — a **stacked** branch.

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
   `tasks/<id>.json` (so the Implement phase, PR creation, and rebases reuse the
   exact same base). Use the superpowers `using-git-worktrees` skill. NEVER work on
   `main`, and never branch a task off `main`.

2. **Verify-and-extend the planner's brief — do NOT re-derive it.** The cycle
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
   - **Determine this task's `complexity`** (`high|medium|low|trivial`, from the
     verified touch set + shared surfaces) and write it to `tasks/<id>.json` — the
     orchestrator reads it to pick the Implement model, and the Implement phase
     reads it to pick the pre-push review depth. **`trivial`** is reserved for a
     change with **no logic/behavior change** — a typo, lint/formatting fix, or
     comment/doc-only edit; anything that alters behavior is at least `low`.

   Then: `high`/`medium` → write the plan, set `status = specified`, return.
   `trivial`/`low` → set `status = implementing` and continue below (fused path).

## Implement phase (TDD → PR; spawned at modelPolicy[complexity], or fused)

3. **Implement (TDD).** Execute the plan via `executing-plans` /
   `subagent-driven-development` with `test-driven-development` — for a
   `trivial`/`low` fused run, implement directly with TDD, no sub-orchestration.
   Honor all repo `CLAUDE.md` rules and invoke required project skills
   (migrations, etc.).
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
   bar as PR comments): fix the real ones and re-run the gate; a clean/decline-only
   pass just proceeds. This self-review is *before the PR exists* and is distinct
   from the monitor/fix loop that answers reviewer comments on an already-open PR.
5. **PR (one per task, based on this task's `baseBranch`).** Commit on *this
   task's* branch, `git push -u origin <branch>` (this branch only, never main,
   never another task's branch), and open **this task's own** PR **targeting its
   `baseBranch`**:
   `gh pr create --base <baseBranch> --head <branch> --title "<title>"` — that base
   is the integration branch (0/2+ blockers) or the single blocker's branch
   (stacked). Resolve `<title>` via the **PR title convention** (below). Never
   `--base main` for a task. Do not append your changes onto a sibling task's
   branch/PR (unless this task qualifies for the trivial-fold exception, which the
   top orchestrator decides — a task agent always defaults to its own PR).
   **Open as `--draft` unless it is already safe to merge**: any stacked PR, or any
   PR whose CI hasn't gone green yet, is created `--draft` so a human can't merge
   it out from under an unmerged base or in-flight work; un-draft (`gh pr ready`)
   only once settled and its base has merged.
   **Scale the PR body to `complexity` (see PR content requirements, below).**
   Write `prNumber/Url`, `branch`, `baseBranch`, `worktreePath`, `isDraft`,
   `decisionLog`, `status = open` to `tasks/<id>.json`; return the summary. You end
   this tick here (do NOT busy-wait for review).
6. **Cleanup (your last act, once the human merges the task PR into its base).**
   On merge: `git worktree remove --force .claude/worktrees/<branch>`, delete the
   local branch, prune. Set `status = done` in `tasks/<id>.json`, sync the tracker
   (sub-issue → Done). **You die** — the orchestrator stops re-spawning you. (The
   integration branch + plan PR live until the human merges the plan PR into
   `main`.)

## Monitor pass (on your own settled PR, status = open)

Runs only when the task is **idle and `status = open`** (PR up, no agent was in
flight). Never during a build or while another fix round-trip is in flight. Runs in
the task's own worktree; writes its own `tasks/<id>.json`. For this task's PR `<n>`:

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
       reviews(last:30){nodes{author{login} state body submittedAt}}
       comments(last:100){nodes{databaseId author{login} body url createdAt}}
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
   review (a `COMMENTED` review carries feedback too), every general conversation
   comment, every inline thread with `isResolved` (an **unresolved thread is an
   unaddressed item even if you replied before**), and the full CI rollup. Fall
   back to the individual `gh pr view` / `gh api` REST calls only if the GraphQL
   call fails. You may NOT report a PR as "green / ready / awaiting merge" unless
   this tick you confirmed **all three**: `reviewDecision` is `APPROVED` (or null
   with no requested reviewers), CI rollup all green, and `mergeable` clean.
   Otherwise report the true state and act on it.
1. **Merged?** If `mergedAt` set → do **Cleanup** (step 6 above), set
   `status = done` in `tasks/<id>.json`, and **you die**.
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
   > reviewer like **Macroscope** "doesn't understand the review was solved")
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

   An item is "addressed" once it has been **judged, fixed-or-declined, replied to,
   AND its resolution signaled** (fixed → thread resolved + re-request; declined →
   reply only, thread left open). A posted reply alone is NOT "addressed" for an
   agreed fix. Record `{commentId, replyUrl, threadResolved}` in `answeredComments`
   ONLY after the reply is confirmed posted — never on push alone, so you never
   miss or double-post. Set `status = fixing` only while an *agreed* change is in
   flight, back to `open` once pushed, replied, **and resolution signaled** for
   every touched item.
3. **CI red?** Read the `statusCheckRollup` from step 0; if any check is
   failing/pending, `gh pr checks <n>` for logs, fix in the worktree, push, then
   **`gh pr comment <n> --body '…'`** describing the failure + the fix + commit
   SHA, and record its URL. A CI fix without a posted comment violates NO SILENT
   FIXES.
4. **Merge conflict / behind base?** Rebase the branch onto **this task's
   `baseBranch`** (from `tasks/<id>.json`) — the integration branch
   (`origin/cadence/<slug>-integration`) for a 0/2+-blocker task, or the
   **blocker's branch** for a stacked task; for the **plan PR** it's `origin/main`.
   Never rebase onto `main` for a task. Resolve, force-push the branch (never
   main). **Because the cycle flows instead of gating, a base advancing is
   expected, not exceptional:** when your blocker's branch gets new commits (or
   merges into integration and its branch is deleted — then re-base onto
   integration and update `baseBranch`), rebase onto the new base and keep going.
   **Post a `gh pr comment`** noting the rebase (and call out any behavior
   change), and record its URL.
5. Write `lastCheckedAt` and the updated `status` to `tasks/<id>.json`, run the NO
   SILENT FIXES assertion, then **return** your summary `{id, status, prNumber,
   merged?, renamedTitle?, prState, note}`. `prState` is the TRUE state observed
   this tick — one of `changes_requested | awaiting_review | ci_red | conflict |
   approved_green_awaiting_merge | merged` — so the orchestrator/console reports
   reality (e.g. "awaiting review", not "awaiting merge"). Only emit
   `approved_green_awaiting_merge` when reviewDecision=APPROVED **and** CI green
   **and** mergeable clean. **Never click merge** — leave that to the human.

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
| → `merged` (task PR merged into its base) | Move sub-issue to **Done/Merged**; comment "merged into `<baseBranch>` (cycle plan PR #<n>)" |
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
