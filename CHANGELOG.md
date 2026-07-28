# Changelog

Releases of the plugins in this marketplace. Versions are **per plugin** and tagged
`<plugin>-v<MAJOR.MINOR.PATCH>`.

**One version per shipped batch, not per edit.** Work in progress accumulates under
`Unreleased`; the version and its tag are minted when the batch is handed over. See
"Versioning policy" in `CLAUDE.md`.

## Unreleased

_(nothing yet — next batch lands here)_

---

# cadence

## 3.3.0 – 3.10.0 — 2026-07-28

One session of fixes driven by feedback from a live cycle. **These were released as
eight separate versions in a single afternoon** — that was version inflation caused by a
"bump on every enhancement" reading of the policy, which has since been rewritten to one
version per shipped batch. The work itself is listed below; **none of it had been
exercised by a real cycle at release time**, and the next real signal is a cycle report
run against `3.10.0`.

### Flow and topology

- **`3.3.0`** — a task with 2+ blockers no longer waits for them to merge: it builds a
  **join branch** (integration + every blocker) and starts immediately. A per-tick
  **flow audit** classifies why each unfinished task isn't advancing and treats
  "waiting for a merge" as a defect to convert on the spot.
- **`3.9.0`** — **nothing ever merges into a join branch.** `3.3.0` had task PRs
  *targeting* their join; since a join merges nowhere, merged work was stranded off
  integration and needed a rescue PR (observed live). Tasks now branch *off* the join
  but their PR targets integration; the join is per task, never shared, and is simply
  deleted once its blockers land.

### Conflicts and PR health

- **`3.5.0`** — conflicts stopped going unnoticed. The change-detection call now
  snapshots **base ref heads** (when a human merges, a dependent's own PR fields don't
  change — only its base does), treats `mergeable: UNKNOWN` as unresolved rather than
  "no news", makes conflicted PRs top priority, and fans a merge out to all dependents
  in the same tick. Base syncs **merge the base in** — never a rebase, never a
  force-push on an open PR, so in-flight review threads survive. Added the **scope
  check**: after a sync, the diff must contain only the task's own work (the
  squash-merged-parent trap).

### Review, decisions, and merging

- **`3.3.0`** — **open decisions**: a question the run can't settle becomes an
  answerable numbered comment on the PR with options and a default in effect, blocking
  readiness and auto-merge until answered. **Draft** was redefined as *not reviewable
  yet* (stacking is no longer a reason to hide a PR), and readiness became the skill's
  own checklist rather than a question to the user. **Approval-authorized auto-merge**:
  a human approval on a task PR authorizes its merge, guarded by staleness, open
  decisions, CI, mergeability and a landable base. The plan PR → `main` is never
  auto-merged.
- **`3.8.0`** — **policy is pinned at run open.** A mid-run upgrade no longer changes
  the rules under a live cycle: `run.json` records `policyVersion` and an explicit
  `approvalMergePolicy`, and a run with no policy recorded predates the feature and is
  treated as human-only. (Found when a run started under `3.2.1` reported the `3.3.0`
  merge rules after re-entering.)

### Legibility

- **`3.4.0`** — **task ↔ PR identity** on every surface: an identity header as the first
  line of every PR body, a `cadence:<slug>` label on every PR, a task→PR map table in
  the plan PR, and the rule that a task id never appears without its PR number and a
  plain description.
- **`3.9.0`** — the plan PR carries the **cycle plan as diagrams** (waves +
  dependencies, branch topology) above the map table.
- **`3.10.0`** — **PR bodies are durable descriptions, not dashboards.** No file counts,
  test counts, CI status, or review state — GitHub renders those live and the next
  commit falsifies them. Bodies carry intent, scope in words, commands to verify, and
  events with their reasons.

### Evidence and feedback

- **`3.4.0`** — **the cycle report**: append-only event logs written as things happen,
  rendered to `<runDir>/report.md` (and on demand via the new **`/cadence:report`**) —
  outcome, what needed the human and how long it waited, what went wrong *including
  Cadence's own failures*, flow health, cost, timeline. Plus the **evolution loop**: a
  report brought back to this repo starts an interactive round.
- **`3.5.0`** — the report gained a **per-task ledger**. State files are overwritten in
  place, so every mutation now emits an event (`state.changed`, `snapshot.delta`,
  `base.moved`, …): *if you wrote a field, you owe an event*.
- **`3.6.0`** — run paths announced at open and footered every turn; earned-only desktop
  notifications (cycle complete, a new blocking decision, plan PR ready, a failed task).

### Planning

- **`3.7.0`** — **prep bundles**: the planner folds trivial same-class work into one
  task (`docs` / `config` / `schema`, never mixed), capped, with every scope keeping its
  own requirement ids — so a doc fix doesn't cost a worktree, a spec agent, a PR and a
  merge click.

## 3.2.1 and earlier

See the git history (`git log --oneline`) and the tags `cadence-v3.2.1`,
`cadence-v3.2.0`, …
