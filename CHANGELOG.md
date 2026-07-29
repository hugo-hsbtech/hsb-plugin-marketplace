# Changelog

Releases of the plugins in this marketplace. Versions are **per plugin** and tagged
`<plugin>-v<MAJOR.MINOR.PATCH>`.

**One version per shipped batch, not per edit.** Work in progress accumulates under
`Unreleased`; the version and its tag are minted when the batch is handed over. See
"Versioning policy" in `CLAUDE.md`.

## Unreleased

### cadence — the log the report is rendered from

Feedback from the second real cycle (`voucher-phase2-c2-statements-dashboards`, 16 tasks,
16 merged, 0 failed, 4h 16m). The cycle **opened under 3.6.0** and was upgraded to
3.11.0 mid-run, so most of its capture complaints were already fixed before the report
was written — see `docs/feedback/20260729-second-cycle-report.md` for what was declined
and why. What survived verification is five defects, all of which corrupted either the
run's durable state or the report rendered from it.

- **The event-kind vocabulary is closed, and it is now visible where events are written.**
  The orchestrator invented `cadence.defect`, `merge.executed`, `policy.skew` and
  `wave.dispatched` — all four already had canonical kinds — and mislabelled two more.
  Because the report looks events up by `kind`, an invented kind is a *lost* event: nine
  recorded Cadence defects, a merge under a user grant and a mid-run version skew all
  rendered `not captured` from a log that contained them. `SKILL.md` now carries an
  inline roster of the **orchestrator's own** kinds (the reference was only opened at
  render time, too late to help), a hard "never invent a kind — fall back to `incident`
  with a sub-`kind`" rule, and a "make the kind match the detail" rule. Same for the task
  agent in `task-agent.md`.
- **Every timestamp comes from `date -u`.** An entire run's orchestrator- and
  agent-written timestamps were local time suffixed `Z` (−3h off GitHub's): internally
  consistent, absolutely wrong, and it made every per-task duration unusable — several
  tasks recorded a `readyAt` that postdated their own `mergedAt`. Every `ts` and every
  timestamped state field now comes from `date -u +%Y-%m-%dT%H:%M:%SZ`; GitHub's stamp
  wins where both exist; and the renderer sanity-checks for impossible orderings and
  emits `not captured` rather than quietly reconciling the two clocks with an offset.
- **A detected merge is a delta, not a conclusion.** Four tasks were marked `done` on
  merge-detection with no agent spawned at all — worktrees left on disk,
  `handoff.delivered` never captured for their dependents, and one task file still
  reading `status: "open"` hours after its PR merged. Twice, a cleanup agent was spawned
  *alongside* the task's still-live agent and both wrote the same file. The orchestrator
  now never writes a task file or retires a task itself: a merge spawns exactly one
  monitor agent, which does cleanup and writes the file. **Idle-gating has no exception
  for cleanup.**
- **The orchestrator must accept an empty branch as a valid base.** "Branches are
  published at spec start" was written only to the task agent, so the other half broke:
  an agent pushed an empty branch precisely to open the gate, the orchestrator judged it
  unsafe, and a `low`-complexity task's Opus spec was serialised behind its blocker's
  entire implement + gate + self-review + PR write-up. "Exists" now explicitly means the
  ref is on the remote — not that it has commits.
- **"State the rule, never the derived number" now covers stale observations.** The
  orchestrator told a task the compose `db` container was running: true when measured,
  false when used. A number you computed and a state you observed fail identically, so
  the invariant now names both.

---

# cadence

## 3.12.0 — 2026-07-29

Feedback from running `3.11.0` on a live cycle. Baseline: `3.11.0`.

> One of the three items below is a **fix to a regression `3.11.0` introduced** — its
> branch-publication change removed a real freeze but let dependents start before their
> blockers had decided anything. The other two are the capability that was missing
> underneath it (knowledge transfer between agent contexts) and a planner default
> confirmed by five cycles of the user writing it by hand.

### cadence — sequencing, the handoff, and a default exit gate

Feedback from running **3.11.0**: it started dependent tasks in parallel with the tasks
they depend on. *"One PR implementation needs the knowledge the dependent task — as they
are being executed in parallel, how could the PR that depends know what to be done since
the dependent PR is not done yet?"*

That is a regression 3.11.0 introduced, and the diagnosis is narrow: 3.11.0's producer
check lived **only in the implement phase**. The **spec phase had no gate at all** — and
spec is what decides the plan, the complexity, and (for `trivial`) fuses straight to a
PR. So a dependent's `verify-and-extend` ran against a tree without its blocker's work,
planned from the brief alone, and built on an interface nobody had decided.

- **Two sequencing gates, per edge.** A dependent's **spec** waits until every blocker is
  `specified` (its handoff contract written); its **implement** waits until every blocker
  is `open` (PR up, gate green, self-reviewed). Both clear on another **agent's**
  progress, never on a human's merge — that is the line between sequence and a freeze.
  `awaiting-blocker-spec` / `awaiting-blocker-pr` are legitimate; `waiting-for-merge`
  remains a defect. Independent tasks still run fully parallel.
- **`trivial` may only fuse** spec→implement when its blockers are already `open`.
  Fusing past a sequencing gate is how a dependent ships against code that doesn't exist.
- **The handoff — knowledge that crosses agent contexts.** An agent is spawned fresh,
  does one step, and dies; the planner's brief is a *prediction* written before any code
  existed. Every task now writes `handoff.contract` at the end of spec (exact surfaces:
  names, signatures, routes, entities, migrations — plus decisions and deviations from
  the brief) and `handoff.delivered` when its PR opens (what actually shipped, drift from
  its own contract, and the gotchas a consumer would otherwise rediscover). **The
  orchestrator pastes every blocker's handoff verbatim into the dependent's brief**, and
  **the handoff outranks the brief** where they disagree. A missing or divergent surface
  at implement time is a reported `contract.mismatch`, never something to stub around.
- **Branch publication is retained but demoted.** Publishing at spec start still removes
  the mechanical wait for a branch to exist; it is no longer treated as permission to
  start early.

**Planner: every cycle ends with an exit-gate task, by default.** Each task is verified
against its own base; nothing verified the union — the interaction of parallel work, and
the suites that don't run per-PR (a cycle once merged a migration that failed on any
fresh database, because the only suite that would have caught it runs on a manual
workflow input). The planner now appends a final task depending on every other: the
repo's complete gate plus the cycle's end-to-end promise through production code paths,
owning the cycle-level acceptance criteria no single task owns, **tests and harness
only** — a needed production change is a finding, not something to absorb. It is the one
task allowed an empty R-id list, and a source plan that already ends with an equivalent
phase is adopted rather than duplicated. Pre-write gate assertion 6 enforces it.

## 3.11.0 — 2026-07-29

Three batches, one release — the first driven by evidence from a **completed** cycle
(`voucher-phase2-c1-exceptions-foundations`, 18 tasks, PRs #31–#55) plus a second cycle
caught mid-flight. Baseline: `3.10.0`.

> **Honest status:** `3.10.0` shipped unexercised, and most of this batch is likewise
> unproven — the exception is the branch-publication fix, which was diagnosed against a
> live run's own turn summary. The next real signal is a cycle report run against
> `3.11.0`. Two of the fixes below are corrections to rules that contradicted their own
> invariants in `3.10.0`.

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

### cadence — executor: branches publish at spec start (the real freeze)

The merge-wait rules were all in place and being followed — and cycles still ran their
dependency chains strictly end to end. The cause was a second gate that every
anti-freeze rule permitted by name.

A task's branch was pushed in exactly one place: **PR creation**, at the end of Implement
step 5. Dispatch requires a blocker's branch to *exist on the remote*. So a dependent
waited for its blocker's spec **and** TDD build **and** lint/format/tests **and**
complexity-scaled self-review **and** PR body — everything — before it could start. The
flow audit classified that as `waiting-for-blocker-branch` → **legitimate**. Caught live
in a running cycle: *"Merged (9): T1–T9 · Implementing: T10 · Pending (6): T11–T16 …
the tail is serial by design."* Six tasks idle behind one, waiting on a branch that
already existed in a worktree and simply hadn't been pushed.

- **Publish the branch at spec start** — pushed empty, before a line of code, the moment
  the worktree exists. An empty branch is identical to its base, has no PR and is not
  reviewed; it costs a ref and makes every dependent dispatchable **one tick after the
  blocker is dispatched** rather than after it finishes. Every task's Opus spec phase now
  runs in parallel across the whole cycle.
- **Push commits as you implement**, not at PR-creation — always immediately after
  landing a surface a dependent consumes. The gate protects the PR, not the branch.
- **Dependents check their producers, and never wait.** An implement agent syncs its base
  and verifies the surfaces it consumes are present. If one isn't yet, it builds
  everything else, pushes, records `implementBlockedOn`, and returns to `specified`.
- **The re-spawn is free.** `refSnapshot` already watches every task branch, so the
  blocker's next push moves its ref — a delta that fans out to every dependent in the
  same tick. No polling anywhere.
- `waiting-for-blocker-branch` is now a **defect** when the blocker has already been
  dispatched, and never force-push (a dependent may be based on what you pushed).
- New events `branch.published` / `blocked.producer`; new state `branchPublishedAt` /
  `implementBlockedOn`; two new flow-health rows so the next report shows whether
  branches are being published on time.

Round recorded in `docs/feedback/20260729-first-real-cycle-report.md`, including the six
findings deliberately **not** acted on and the field in the next report that will show
whether each fix worked.

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
