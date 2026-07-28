# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Claude Code plugin marketplace**, not an application. It ships one plugin (`cadence`) made entirely of Markdown — there is no source code, build, lint, or test step. "Building" here means editing skill/command/reference `.md` files and their JSON manifests. Validation is by reading the prose for consistency, not by running a tool.

## Layout

```
.claude-plugin/marketplace.json      marketplace manifest → lists the cadence plugin, source ./plugins/cadence
plugins/cadence/
  .claude-plugin/plugin.json         plugin manifest (name/version/author)
  commands/                          slash-command entrypoints (thin — they load a skill)
    plan.md                       /cadence:plan        → loads cadence-planner
    ship.md                /cadence:ship → loads cadence-executor
  skills/
    cadence-planner/SKILL.md           the planner behavior + references/cycle-plan-template.md
    cadence-executor/SKILL.md          the TOP ORCHESTRATOR behavior + references/{task-agent,execution-state,pr-template}.md
```

The executor's behavior is deliberately split in two: `SKILL.md` is the **top
orchestrator's** playbook (preflight, dispatch, change detection, re-arm), while
`references/task-agent.md` is the **per-task agent's** playbook (spec/implement
phases, Monitor pass, JUDGE BEFORE YOU ACT, NO SILENT FIXES, PR conventions, tracker
sync) — every spawned agent is told to read it first. This keeps the orchestrator's
per-wakeup context small. When editing executor behavior, put it in the right file
and keep the SKILL.md one-paragraph summary of the agent in sync.

Commands are deliberately thin: each one resolves its `$ARGUMENTS` (a task list, a Linear/Jira key, or a plan-doc path) and hands off to the matching skill. The behavior lives in the `SKILL.md` files; the `references/*.md` are the templates and schemas those skills read at runtime.

## The two-skill pipeline (the core architecture)

The plugin implements a **plan → execute** handoff for parallel development cycles. Understanding the contract between the two skills is the whole point:

1. **`cadence-planner`** ingests tasks, runs deep parallel codebase analysis to compute each task's *touch set* (creates / edits / reads / shared surfaces), builds a dependency graph (declared deps + producer→consumer + write-write conflicts), and topologically levels tasks into parallel **waves**. It is **plan-only** — it writes a cycle-plan doc to `docs/plans/proposed/<YYYYMMDD-HHMM>-<slug>-<task-id>.md` and stops. It never implements or dispatches.

2. **`cadence-executor`** consumes that plan doc and autonomously drives every task to a merged PR, then keeps monitoring open PRs on a `ScheduleWakeup` loop until a **human** merges.

The handoff is by **file + metadata header**, not by memory: the planner records `Slug` and `Task-id` in the plan's metadata block, and the executor reads them from content (the filename is timestamped and not authoritative).

## Executor invariants (the rules that must never be broken when editing cadence-executor)

These constraints define the executor and appear throughout its `SKILL.md`. When editing, preserve them exactly:

- **Stacked branch topology, and it never gates on merges.** One **integration branch** `cadence/<slug>-integration` holds the plan docs as a *plan PR → `main`* (opened draft). A task's PR base is derived from its blockers and is never `main`: **0 blockers** → integration; **1 blocker** → that blocker's branch (stacked); **2+ blockers** → its own **join branch** `cadence/<slug>-t<id>-join` (integration with every blocker's branch merged in), built by the task's own agent so the task starts while its blockers are still open, refreshed as they advance, and retired (PR re-targeted to integration, branch deleted) once they've all landed. Each task PR lands in integration; the plan PR into `main` is merged last, by the human.
- **Flow audit every tick — "waiting for a merge" is a defect.** Each tick classifies why every unfinished task isn't advancing. Agent-in-flight, awaiting human review, awaiting a human decision, blocked by a failed dep, and waiting for a blocker's *branch* to appear are legitimate; being held because another PR hasn't **merged** is not — it must be converted the same tick (build/refresh a join, rebase, re-target). Three consecutive ticks on the same non-dispatchable reason lands in `run.json.stalls` and the turn summary.
- **Sweep generated docs off `main`.** The planner writes the cycle-plan/design docs into the **`main` working tree**, where they sit *uncommitted/untracked*. When the executor creates the integration branch (as a **worktree**), it **moves** this cycle's in-scope docs (plan doc + this run's `docs/plans/proposed/` and `docs/superpowers/specs/` files) into that worktree, commits them there, and asserts the `main` checkout is left clean of cycle files. Out-of-scope uncommitted changes (the user's unrelated WIP) are never touched. This is what stops cycle files from stranding on `main`.
- **Never push/commit to `main`; never merge the plan PR.** The plan PR → `main` is the human's gate, unconditionally — no policy or approval hands it over.
- **Approval-authorized auto-merge (task PRs only).** An approving review by a **human** on a task PR is that human's authorization for the task's own agent to merge it into its base. It merges only when every guard holds in the same monitor tick: human (not bot) approval; approval not dismissed or superseded by `CHANGES_REQUESTED`; **the approval still describes the code** (head identical to `approvedSha`, or differing only by a pure rebase or a `trivial`-class comment/doc/formatting edit — anything else is stale, and when in doubt it is stale); **no unresolved `pendingDecisions`**; every comment answered and every agreed thread resolved; CI fully green; `mergeable` clean; and **the base is landable** — never a parent task's branch whose PR is still open (that would inject unreviewed commits into the parent PR) and never a join branch (retire it to integration first), reported as `approved_awaiting_base`, which is a waiting *button*, not a frozen task. It posts the authorization record on the PR **before** merging (**NO SILENT MERGES**) and records `autoMerge` in the task file. Policy knob: `run.json.approvalMergePolicy = "auto"` (default) | `"off"`; the separate explicit user grant (`mergeAuthorization`) still applies to its named scope.
- **Draft ⇔ not reviewable, and readiness is the skill's own call.** A PR is draft only while an agent is in flight, the gate/CI is red, the self-review hasn't run, the body is unwritten, or a **blocking open decision** is unanswered. **Being stacked on an unmerged base is NOT a reason to stay draft** (that's what stranded whole cycles in draft) — stacking is stated in the PR body instead. The readiness checklist runs at PR-open and every monitor tick, and the agent un-drafts itself when it passes. **Never ask the user whether to mark PRs ready**, and never report a cycle as "ready for review" while any PR is a draft.
- **Open decisions must be actionable, never silent.** A choice that can't be settled on the merits (product/policy intent, ambiguous criteria, irreversible or security-relevant choices) is not a buried default: it becomes a numbered `D<n>` question posted on the PR with real options, the provisional default in effect, and an answer protocol; pinned at the top of the PR body; recorded in `tasks/<id>.json.pendingDecisions`; mirrored into `run.json.attention`; and (when `blocking`) it keeps the PR draft and disqualifies auto-merge until answered. Answers are harvested each monitor tick. If a human merges over an open decision anyway, it is carried into `run.json.unresolvedDecisions` and the final summary. Everything settleable stays autonomous — a decision-log line, not a question.
- **Every turn ends with an honest report.** Per-PR true state (draft *and why*, awaiting review, awaiting decision, approved, auto-merged, merged) plus the `attention` list — every open decision with its answer link, parked review loops, failed tasks, and the plan PR once ready.
- **One dedicated agent and one PR per task.** Never combine tasks into a shared branch/PR. Branch names are descriptive: `cadence/<slug>-t<id>-<task-slug>`.
- **Explicit dependencies at preflight.** The **superpowers plugin is a hard requirement** — the preflight gate STOPS the run with install instructions if its skills aren't available (per-task agents invoke `superpowers:*` throughout). **graphifyy is optional**: detected at preflight (`preflight.graphify = "ok" | "absent"`), used to ground analysis in graph queries when present, silently skipped when absent — it must never become a blocker.
- **Two-level, re-spawn-each-tick execution.** A *thin top orchestrator* (the main loop) only schedules/gates; it spawns one **per-task agent** per *idle active* task with work each tick. Agents are re-spawned every wakeup rather than kept alive, because monitoring spans days and an agent lives only one turn — continuity is the durable worktree + task file, not the process.
- **Idle-gating + change detection.** Only act on a task with `agentInFlight = false`. A task whose agent is mid-round-trip (`specifying`/`implementing`/`fixing`) is skipped; a settled `open` PR gets a monitor tick **only when the orchestrator's one batched read-only GraphQL snapshot (`run.json.prSnapshot`) shows a delta** — no delta, no spawn, with three narrow exceptions that are real work rather than polling: its readiness or auto-merge guards could now pass, its base advanced (rebase / join refresh / join retirement), or an open decision is due its one reminder. Change detection is strictly read-only: the orchestrator never triages, replies, or acts on PR content.
- **Adaptive monitor cadence.** The `ScheduleWakeup` interval is `monitorBackoff`-driven: base 180s while hot (deltas / spawns / agents in flight), doubling per fully-quiet tick to a `maxSeconds` cap (default 1800); any activity resets it. Never a fixed frenetic interval, and never an external scheduler.
- **Model-by-phase, with a fused fast path.** Spec/analysis is always **Opus, high effort**; implementation is Opus or Sonnet by the `complexity` the spec phase determined; monitor/fix/cleanup is **Sonnet, low**. Complexity is `high | medium | low | trivial`, decided during the spec phase, not at plan time. A spec agent that decides `trivial`/`low` **continues straight into implementation in the same invocation** (one spawn, no context re-derivation); only `medium`/`high` stop at `specified` for a separately-tiered implement agent.
- **Spec consumes the planner's brief — verify-and-extend, never re-derive.** The plan's per-task context brief (touch set + requirements + acceptance criteria) is the spec phase's starting point; it is verified against current code (graphify-first when available) and extended on drift, with sub-subagents spawned only for genuine unknowns. Re-running full from-scratch analysis per task is the double-payment this rule exists to kill.
- **Complexity-gated pre-push self-review.** Before opening a task PR (after the green lint/format/tests gate), the implement agent runs a self `/code-review` scoped to *its own branch diff*, scaled to `complexity`: **`trivial` skips it** (a typo/lint/doc change with no behavior change — the gate is enough, don't make it bureaucratic); `low`/`medium` → `/code-review low`; `high` → `/code-review high`. This is a pre-PR self-review, separate from the JUDGE-BEFORE-YOU-ACT monitor loop that answers reviewer comments on an open PR.
- **Bounded review loops (REVIEW CONVERGENCE BOUND).** Both review loops have hard convergence bounds so a cycle can never stall in endless reviews: the pre-push self-review runs **exactly once** (one review, one fix pass — after fixes only the lint/format/tests gate is re-run, never a second `/code-review`); the monitor loop allows at most **3 fix→re-request rounds per reviewer** (tracked in `tasks/<id>.json.reviewerFixRounds`) — after that the agent stops re-requesting that reviewer, posts one summary comment, and parks the remaining findings for the human, letting the PR go quiet so the adaptive backoff engages. Per reviewer, not per PR: humans and other reviewers are unaffected. Neither cap ever blocks fixing genuinely broken code (red CI, conflicts, real bugs).
- **JUDGE BEFORE YOU ACT + NO SILENT FIXES.** Reviews are suggestions to evaluate on the merits (agree / alternative / decline / clarify), never commands. Every code change made in response to a comment/CI/conflict must get a real, *verified-posted* `gh` reply (capture the URL) before it counts as handled; agreed-and-fixed items also resolve their review thread and re-request the reviewer.
- **Turn-end invariant.** If any task is still active or the plan PR isn't merged, the last action of the turn **must** be a `ScheduleWakeup` (at the current adaptive interval) re-entering `/cadence:ship <plan-path>`. Monitoring uses only in-session `ScheduleWakeup` — never cron or any external/paid scheduler.

## State model (per-run directory, split by owner)

Executor state lives under `.cadence/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/` (add `.cadence/` to the target repo's `.gitignore`; never commit cycle state). Ownership is strict and is the reason state is a directory rather than one file — parallel agents would clobber a shared JSON:

- `run.json` — **orchestrator-owned** (only writer): roster, wave schedule, preflight gate (incl. `plugins.superpowers` / `graphify`), integration/plan-PR pointers, `prTitlePattern`, `modelPolicy`, `agentInFlight` flags, `prSnapshot` change-detection baselines, `monitorBackoff` adaptive-interval state, `approvalMergePolicy`, `attention` (the rebuilt-every-tick "needs you" list), `stalls`, `unresolvedDecisions`.
- `tasks/<id>.json` — **task-agent-owned** (each task's agent is the sole writer): that task's status, branch + `baseBranch`/`joinBranch`, worktree, PR number, `isDraft`/`draftReason`/`readyAt`, `pendingDecisions`, `approvals` (per-reviewer, with the `approvedSha` the auto-merge staleness test anchors on), `autoMerge`, `answeredComments` (with `replyUrl` proof), and `decisionLog`.

Task status flow: `pending → specifying → specified → implementing → open → (fixing ↔ open) → merged → done` (or `failed`); the fused fast path for `trivial`/`low` complexity goes `specifying → open` directly (the spec agent implements in the same invocation). `open` spans draft and ready — the draft flag is re-evaluated by the readiness checklist every tick, never by asking the user. Full schema and field notes: `plugins/cadence/skills/cadence-executor/references/execution-state.md`.

## Versioning policy (bump on every AI enhancement)

**Every time an AI agent enhances this project, it MUST bump the semantic version in `plugins/cadence/.claude-plugin/plugin.json` as part of the same change** — never leave an enhancement unversioned. The bump size follows the nature of the change (`MAJOR.MINOR.PATCH`):

- **PATCH** (`x.y.Z`) — wording clarifications, typo/formatting fixes, tightening prose, or reference/template edits that do **not** change any skill's behavior or the planner↔executor contract.
- **MINOR** (`x.Y.0`) — backward-compatible additions: a new skill or command, a new optional plan-doc/state field, or added behavior that doesn't break existing plans or state files.
- **MAJOR** (`X.0.0`) — breaking changes: renaming/removing a skill or command, or any change to the plan-doc metadata header, filename convention, wave/touch-set fields, or state schema that would break plans or `.cadence/` runs produced by an earlier version.

When a single change spans several types, bump by the highest that applies. State the resulting version in the change summary. If the future repo adds a `CHANGELOG.md`, record the bump and its reason there too.

**Every version bump MUST also create a git tag** on the commit that contains the bump, and the tag MUST be pushed together with the commit. Tags are plugin-scoped (the marketplace can host several plugins, each with its own version): `<plugin>-v<MAJOR.MINOR.PATCH>`, e.g. `cadence-v3.2.0`. Use an annotated tag whose message is a one-line summary of the change (`git tag -a cadence-v3.2.0 -m "…"`), then `git push origin <tag>` (or push with `--follow-tags`). A bump without its tag is incomplete — if you find a bumped version with no matching tag, backfill the tag on the commit that introduced that version.

## Documentation per plugin (philosophy)

**Every plugin ships its own deep-dive doc at `plugins/<name>/README.md`** — the comprehensive, human-facing reference for that plugin (concepts, worked examples, diagrams, command reference, state model, FAQ). The repo-root `README.md` stays a thin marketplace index that points into each plugin's doc. The per-plugin README is *documentation only*: the `SKILL.md`/reference files remain the source of truth for behavior, so keep the doc in sync when behavior changes — but editing the doc alone does **not** require a version bump (it changes no skill behavior or contract). Cadence's is `plugins/cadence/README.md`.

## Conventions when editing skills

- **Keep commands thin, behavior in skills.** A `commands/*.md` file resolves arguments and loads a skill; don't duplicate skill logic into it.
- **Templates are references, not prose.** PR body shape lives in `references/pr-template.md`; plan-doc shape in `cadence-planner/references/cycle-plan-template.md`; state schema in `references/execution-state.md`. Edit the reference, and keep the SKILL's inline summary in sync with it.
- **The planner/executor contract is load-bearing.** If you change the plan-doc metadata header, filename convention, or wave/touch-set fields, update *both* skills — the executor parses what the planner emits.
- Preserve the frontmatter `name`/`description` in every `SKILL.md`; the `description` is what triggers the skill and is intentionally verbose.
