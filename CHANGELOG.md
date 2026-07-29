# Changelog

Releases of the plugins in this marketplace. Versions are **per plugin** and tagged
`<plugin>-v<MAJOR.MINOR.PATCH>`.

**One version per shipped batch, not per edit.** Work in progress accumulates under
`Unreleased`; the version and its tag are minted when the batch is handed over. See
"Versioning policy" in `CLAUDE.md`.

## Unreleased

### cadence — planner: plan-doc fidelity

From two live cycles in `pagana-catalog-apps` — a rich plan (14 tasks, 481 lines, 137
requirement ids, balanced ledger) followed by a poor one (16 tasks, 178 lines, **zero**
requirement ids, no ledger, ASCII graph, prose briefs). The instructions for all of it
already existed; nothing forced them.

- **Read the template before writing.** Step 5 now hard-requires reading
  `references/cycle-plan-template.md` in full, and says plainly that the Output-shape
  summary is an index, not a substitute.
- **A brief is a field set, not prose** — Goal/done · Requirements covered · Creates ·
  Edits · Reads · Shared surfaces · Blocks · Notes. Prose briefs break the requirement
  ledger, the executor's scope check, and verify-and-extend. Hard-won detail belongs in
  **Notes**, in addition to the fields.
- **Pre-write gate:** five mechanical assertions before the file is written — R-ids per
  brief, a balanced ledger, briefs carrying their fields, a `mermaid` graph with **no
  box-drawing characters anywhere**, and template section order.
- **Never compress the plan to save space** — length scales with task count; a short
  plan for a large cycle is the symptom. Split the cycle instead.
- **Mermaid only**, with a template example that demonstrates what tempted the ASCII:
  wave subgraphs, labelled dashed conflict edges, multi-parent fan-in.

Round recorded in `docs/feedback/20260729-plan-doc-fidelity.md`, including what was
deliberately left unchanged.

### cadence — executor: the first real cycle report

From `voucher-phase2-c1-exceptions-foundations` in `pagana-catalog-apps` — 18 tasks,
PRs #31–#55, plan PR #30 merged at `cb630fd`. The run opened on **3.2.1** and was
reloaded to **3.6.0** mid-run. Every finding was re-anchored against 3.10.0 before being
accepted; several of the report's complaints were already fixed, and two of its biggest
were not.

**Topology and readiness — two defects still live in 3.10.0**

- **A joined task's PR targets INTEGRATION, never the join.** The invariant had been
  rewritten everywhere *except* the two places an agent reads before opening a PR:
  `SKILL.md`'s construction bullet and `task-agent.md`'s `gh pr create` step both still
  said the PR targets the join (`execution-state.md` repeated it a third time). That is
  the exact defect that merged four PRs (#38/#39/#40/#42) into join branches, stranding
  them off integration and costing a three-PR recovery. Fixed at all three sites, with
  the incident quoted at the point of use.
- **Readiness now requires CI TERMINAL and GREEN on the PR's current head SHA.** "CI has
  no failing check" is satisfied by an *empty* check set and by `QUEUED` checks — it
  un-drafted two PRs (#36, #39) onto code nothing had verified. The test is positive:
  reported on this SHA, non-empty, all terminal, none failed; otherwise `draftReason:
  ci_pending` and the next tick un-drafts it. A repo confirmed to have no CI is the only
  exemption (`ciExpected: false`).

**Upgrades can no longer re-base a live cycle**

- **Branch topology joins the pinned promises** (merge policy · draft/readiness · `main`
  safety · **topology**). A 3.2.1 → 3.6.0 reload adopted the new base model silently,
  against the plan's §7 *and* an explicit user instruction already in state.
  `run.json.topology` is written at run open; **the plan doc wins** over the skill's
  default when they disagree; a drift on resume keeps the pinned value and logs a
  `topology.skew` incident.
- `/cadence:ship` states that on an already-open run **`run.json` wins over the command
  text** — the pinned policy is not re-litigated on resume.

**Model tiers actually apply now**

- **Only `trivial` fuses.** The fused fast path had the opus/high spec agent implement
  `trivial` *and* `low` tasks in one invocation — and one invocation is one model, so
  every `low` task was **built on Opus** while `medium` built on Sonnet. The cost curve
  was inverted: the cheapest tasks got the most expensive model. `low` now stops at
  `specified` and gets its own sonnet/low implement agent.

**Agent discipline**

- **STATE THE RULE, NEVER THE DERIVED NUMBER.** Briefs carry facts, never counts or
  arithmetic the orchestrator computed. Three wrong numbers reached briefs in one cycle;
  one (`REQUIRED_PATHS` "21 → 23", confusing path keys with operations — the real delta
  was zero) would have shipped a padded array and a false docblock. An agent that finds
  a brief false **declines and reports it**.
- **No agent waits inside a tick** — no sleep, no polling, no retry-until-green. CI
  running *is* the answer: return `prState: ci_pending`. A monitor that looped on CI
  burned ~80k subagent tokens per pass over four passes and needed `TaskStop`.
- **A task agent writes its task file before returning.** One task file said "still
  specifying, nothing merged" while its PR had long since landed.

**Capture — the report's Cost section was almost entirely `not captured`**

- A **`tick` event every wake-up**, quiet ones included, emitted as part of the re-arm
  step (the kind was declared but never instructed anywhere).
- **Every** spawn logs `task.spawn` — ad-hoc repair, verification and housekeeping
  agents included.
- A new **`incident`** event kind + `run.json.incidents`: Cadence's own failures are
  first-class evidence. One run's ledger ended up thinner than its state files, with the
  orchestrator's three errors, a collision and an open risk existing only in `run.json`.

**Two more**

- **`main` is watched.** Its head OID is in `refSnapshot`; a moved `main` triggers a
  merge into integration and a scope check. An external PR merged mid-cycle broke an
  already-merged migration and was only noticed when the plan PR went `CONFLICTING`.
- **One orchestrator per task.** Before a task's first spawn, if its branch or a matching
  PR already exists, don't spawn — surface `possible-concurrent-session`.

**Removed**

- **PR labels, entirely.** Cadence no longer creates a label, applies one, or asks for
  one — the repo's label set belongs to its humans. Identity rests on the PR body's
  identity header, the plan PR's cycle map, and the id+PR+description rule. `cycleLabel`
  is gone from the schema.

Round recorded in `docs/feedback/20260729-first-real-cycle-report.md`, including the six
findings deliberately **not** acted on and the field in the next report that will show
whether each fix worked.

---

# cadence

## 3.10.0 — 2026-07-28

One batch, one release. Everything below came from a single session of fixes driven by
feedback from a live cycle, on top of `3.2.1`.

> **Withdrawn intermediates.** This batch was first published as eight separate versions
> in one afternoon (`3.3.0` … `3.10.0`) — version inflation caused by a "bump on every
> enhancement" reading of the policy, since rewritten to one version per shipped batch.
> The tags `cadence-v3.3.0` through `cadence-v3.9.0` have been **deleted**; those
> versions never represented anything anyone should install, and `3.10.0` is the single
> release for the whole batch. `3.2.1` is its predecessor. The version numbers 3.3–3.9
> are retired and will not be reused.
>
> **None of this had been exercised by a real cycle at release time.** The next real
> signal is a cycle report run against `3.10.0`.

### Flow and topology

- a task with 2+ blockers no longer waits for them to merge: it builds a
  **join branch** (integration + every blocker) and starts immediately. A per-tick
  **flow audit** classifies why each unfinished task isn't advancing and treats
  "waiting for a merge" as a defect to convert on the spot.
- **nothing ever merges into a join branch** — an earlier cut of this batch had task PRs
  *targeting* their join; since a join merges nowhere, merged work was stranded off
  integration and needed a rescue PR (observed live). Tasks now branch *off* the join
  but their PR targets integration; the join is per task, never shared, and is simply
  deleted once its blockers land.

### Conflicts and PR health

- conflicts stopped going unnoticed. The change-detection call now
  snapshots **base ref heads** (when a human merges, a dependent's own PR fields don't
  change — only its base does), treats `mergeable: UNKNOWN` as unresolved rather than
  "no news", makes conflicted PRs top priority, and fans a merge out to all dependents
  in the same tick. Base syncs **merge the base in** — never a rebase, never a
  force-push on an open PR, so in-flight review threads survive. Added the **scope
  check**: after a sync, the diff must contain only the task's own work (the
  squash-merged-parent trap).

### Review, decisions, and merging

- **open decisions**: a question the run can't settle becomes an
  answerable numbered comment on the PR with options and a default in effect, blocking
  readiness and auto-merge until answered. **Draft** was redefined as *not reviewable
  yet* (stacking is no longer a reason to hide a PR), and readiness became the skill's
  own checklist rather than a question to the user. **Approval-authorized auto-merge**:
  a human approval on a task PR authorizes its merge, guarded by staleness, open
  decisions, CI, mergeability and a landable base. The plan PR → `main` is never
  auto-merged.
- **policy is pinned at run open.** A mid-run upgrade no longer changes
  the rules under a live cycle: `run.json` records `policyVersion` and an explicit
  `approvalMergePolicy`, and a run with no policy recorded predates the feature and is
  treated as human-only. (Found when a run started under `3.2.1` reported the `3.3.0`
  merge rules after re-entering.)

### Legibility

- **task ↔ PR identity** on every surface: an identity header as the first
  line of every PR body, a `cadence:<slug>` label on every PR, a task→PR map table in
  the plan PR, and the rule that a task id never appears without its PR number and a
  plain description.
- the plan PR carries the **cycle plan as diagrams** (waves +
  dependencies, branch topology) above the map table.
- **PR bodies are durable descriptions, not dashboards.** No file counts,
  test counts, CI status, or review state — GitHub renders those live and the next
  commit falsifies them. Bodies carry intent, scope in words, commands to verify, and
  events with their reasons.

### Evidence and feedback

- **the cycle report**: append-only event logs written as things happen,
  rendered to `<runDir>/report.md` (and on demand via the new **`/cadence:report`**) —
  outcome, what needed the human and how long it waited, what went wrong *including
  Cadence's own failures*, flow health, cost, timeline. Plus the **evolution loop**: a
  report brought back to this repo starts an interactive round.
- the report gained a **per-task ledger**. State files are overwritten in
  place, so every mutation now emits an event (`state.changed`, `snapshot.delta`,
  `base.moved`, …): *if you wrote a field, you owe an event*.
- run paths announced at open and footered every turn; earned-only desktop
  notifications (cycle complete, a new blocking decision, plan PR ready, a failed task).

### Planning

- **prep bundles**: the planner folds trivial same-class work into one
  task (`docs` / `config` / `schema`, never mixed), capped, with every scope keeping its
  own requirement ids — so a doc fix doesn't cost a worktree, a spec agent, a PR and a
  merge click.

## 3.2.1 and earlier

See the git history (`git log --oneline`) and the tags `cadence-v3.2.1`,
`cadence-v3.2.0`, … Tag list = release list: there is no tag for a version that was
never a shipped batch.
