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
    cadence-executor/SKILL.md          the executor behavior + references/{execution-state,pr-template}.md
```

Commands are deliberately thin: each one resolves its `$ARGUMENTS` (a task list, a Linear/Jira key, or a plan-doc path) and hands off to the matching skill. The behavior lives in the `SKILL.md` files; the `references/*.md` are the templates and schemas those skills read at runtime.

## The two-skill pipeline (the core architecture)

The plugin implements a **plan → execute** handoff for parallel development cycles. Understanding the contract between the two skills is the whole point:

1. **`cadence-planner`** ingests tasks, runs deep parallel codebase analysis to compute each task's *touch set* (creates / edits / reads / shared surfaces), builds a dependency graph (declared deps + producer→consumer + write-write conflicts), and topologically levels tasks into parallel **waves**. It is **plan-only** — it writes a cycle-plan doc to `docs/plans/proposed/<YYYYMMDD-HHMM>-<slug>-<task-id>.md` and stops. It never implements or dispatches.

2. **`cadence-executor`** consumes that plan doc and autonomously drives every task to a merged PR, then keeps monitoring open PRs on a `ScheduleWakeup` loop until a **human** merges.

The handoff is by **file + metadata header**, not by memory: the planner records `Slug` and `Task-id` in the plan's metadata block, and the executor reads them from content (the filename is timestamped and not authoritative).

## Executor invariants (the rules that must never be broken when editing cadence-executor)

These constraints define the executor and appear throughout its `SKILL.md`. When editing, preserve them exactly:

- **Stacked branch topology.** One **integration branch** `cadence/<slug>-integration` holds the plan docs as a *plan PR → `main`* (opened draft). Every task branches **off integration** and every task PR **targets integration**, never `main`. The human merges each task PR into integration, then merges the single plan PR into `main` last.
- **Sweep generated docs off `main`.** The planner writes the cycle-plan/design docs into the **`main` working tree**, where they sit *uncommitted/untracked*. When the executor creates the integration branch (as a **worktree**), it **moves** this cycle's in-scope docs (plan doc + this run's `docs/plans/proposed/` and `docs/superpowers/specs/` files) into that worktree, commits them there, and asserts the `main` checkout is left clean of cycle files. Out-of-scope uncommitted changes (the user's unrelated WIP) are never touched. This is what stops cycle files from stranding on `main`.
- **Never push/commit to `main`; never merge any PR autonomously.** Merging (both task PRs and the plan PR) is always the human's gate.
- **One dedicated agent and one PR per task.** Never combine tasks into a shared branch/PR. Branch names are descriptive: `cadence/<slug>-t<id>-<task-slug>`.
- **Two-level, re-spawn-each-tick execution.** A *thin top orchestrator* (the main loop) only schedules/gates; it spawns one **per-task agent** per *idle active* task each tick. Agents are re-spawned every wakeup rather than kept alive, because monitoring spans days and an agent lives only one turn — continuity is the durable worktree + task file, not the process.
- **Idle-gating.** Only act on a task with `agentInFlight = false`. A task whose agent is mid-round-trip (`specifying`/`implementing`/`fixing`) is skipped; only a settled `open` PR gets a monitor tick.
- **Model-by-phase.** Spec/analysis is always **Opus, high effort**; implementation is Opus or Sonnet by the `complexity` the spec phase determined; monitor/fix/cleanup is **Sonnet, low**. Complexity is `high | medium | low | trivial`, decided during the spec phase, not at plan time.
- **Complexity-gated pre-push self-review.** Before opening a task PR (after the green lint/format/tests gate), the implement agent runs a self `/code-review` scoped to *its own branch diff*, scaled to `complexity`: **`trivial` skips it** (a typo/lint/doc change with no behavior change — the gate is enough, don't make it bureaucratic); `low`/`medium` → `/code-review low`; `high` → `/code-review high`. This is a pre-PR self-review, separate from the JUDGE-BEFORE-YOU-ACT monitor loop that answers reviewer comments on an open PR.
- **JUDGE BEFORE YOU ACT + NO SILENT FIXES.** Reviews are suggestions to evaluate on the merits (agree / alternative / decline / clarify), never commands. Every code change made in response to a comment/CI/conflict must get a real, *verified-posted* `gh` reply (capture the URL) before it counts as handled; agreed-and-fixed items also resolve their review thread and re-request the reviewer.
- **Turn-end invariant.** If any task is still active or the plan PR isn't merged, the last action of the turn **must** be a `ScheduleWakeup` re-entering `/cadence:ship <plan-path>`. Monitoring uses only in-session `ScheduleWakeup` — never cron or any external/paid scheduler.

## State model (per-run directory, split by owner)

Executor state lives under `.cadence/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/` (add `.cadence/` to the target repo's `.gitignore`; never commit cycle state). Ownership is strict and is the reason state is a directory rather than one file — parallel agents would clobber a shared JSON:

- `run.json` — **orchestrator-owned** (only writer): roster, wave schedule, preflight gate, integration/plan-PR pointers, `prTitlePattern`, `modelPolicy`, `agentInFlight` flags.
- `tasks/<id>.json` — **task-agent-owned** (each task's agent is the sole writer): that task's status, branch, worktree, PR number, `answeredComments` (with `replyUrl` proof), and `decisionLog`.

Task status flow: `pending → specifying → specified → implementing → open → (fixing ↔ open) → merged → done` (or `failed`). Full schema and field notes: `plugins/cadence/skills/cadence-executor/references/execution-state.md`.

## Versioning policy (bump on every AI enhancement)

**Every time an AI agent enhances this project, it MUST bump the semantic version in `plugins/cadence/.claude-plugin/plugin.json` as part of the same change** — never leave an enhancement unversioned. The bump size follows the nature of the change (`MAJOR.MINOR.PATCH`):

- **PATCH** (`x.y.Z`) — wording clarifications, typo/formatting fixes, tightening prose, or reference/template edits that do **not** change any skill's behavior or the planner↔executor contract.
- **MINOR** (`x.Y.0`) — backward-compatible additions: a new skill or command, a new optional plan-doc/state field, or added behavior that doesn't break existing plans or state files.
- **MAJOR** (`X.0.0`) — breaking changes: renaming/removing a skill or command, or any change to the plan-doc metadata header, filename convention, wave/touch-set fields, or state schema that would break plans or `.cadence/` runs produced by an earlier version.

When a single change spans several types, bump by the highest that applies. State the resulting version in the change summary. If the future repo adds a `CHANGELOG.md`, record the bump and its reason there too.

## Conventions when editing skills

- **Keep commands thin, behavior in skills.** A `commands/*.md` file resolves arguments and loads a skill; don't duplicate skill logic into it.
- **Templates are references, not prose.** PR body shape lives in `references/pr-template.md`; plan-doc shape in `cadence-planner/references/cycle-plan-template.md`; state schema in `references/execution-state.md`. Edit the reference, and keep the SKILL's inline summary in sync with it.
- **The planner/executor contract is load-bearing.** If you change the plan-doc metadata header, filename convention, or wave/touch-set fields, update *both* skills — the executor parses what the planner emits.
- Preserve the frontmatter `name`/`description` in every `SKILL.md`; the `description` is what triggers the skill and is intentionally verbose.
