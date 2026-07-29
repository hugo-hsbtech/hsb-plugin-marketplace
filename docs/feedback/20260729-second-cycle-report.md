# Feedback round — second real cycle (`voucher-phase2-c2-statements-dashboards`)

**Source:** `<runDir>/report.md` from `pagana-platform/pagana-catalog-apps`, cycle
`voucher-phase2-c2-statements-dashboards`, rendered 2026-07-29 from 459 events
(141 `run.jsonl` + 318 across 16 `T*.jsonl`).
**Run:** 16 tasks planned · 16 merged · 0 failed · 4h 16m wall clock · plan PR
[#56](https://github.com/pagana-platform/pagana-catalog-apps/pull/56) merged (`6cce714`).
**Cadence version:** opened **3.6.0**, upgraded to **3.11.0** mid-run.
**Baseline for this round:** 3.12.0.

> The version history matters more than usual here. This run opened under 3.6.0 — which
> had **no** `tick`, `task.spawn` or `incident` logging rules in the executor SKILL at
> all — and only picked up 3.11.0's rules partway through. Several of the report's
> loudest complaints are therefore complaints about a version that no longer exists.
> They are recorded below as declined, with the verification, rather than silently
> dropped.

---

## Findings

### F1 — Orchestrator invented and mislabelled event kinds · **FIXED**

**Symptom.** The report's own capture-failures section: `incident` never used (nine
Cadence defects and two policy events written to `run.json` and to `cadence.defect` /
`policy.skew` instead); `automerge` never logged (the #65 merge under the user's grant
recorded as a bespoke `merge.executed`); `wave.dispatched` invented where `task.spawn`
existed. Two further events mislabelled: T5's `pr.redraft` was a PR-body correction, and
T9's `gate.failed` has a detail reading *"NOT a failure"*.

**Evidence.** Report §Capture failures; §Feedback → Hurt ("The canonical event vocabulary
isn't visible where the orchestrator writes events"); T5 and T9 per-task ledgers.

**Traces to.** `references/cycle-report.md:60-95` holds the kinds table, and
`SKILL.md` merely pointed at it — a file the orchestrator has no reason to open until
render time, which is exactly when it discovers the vocabulary it spent the run ignoring.

**Class.** Missing capability.

**Changed.** `SKILL.md` (Evidence capture) gains a `THE KIND VOCABULARY IS CLOSED — NEVER
INVENT ONE` block: an inline roster of the **twelve kinds the orchestrator itself
writes**, an explicit statement that the task agents own the rest and it must never write
theirs, the rule that an unrecognised kind is a *lost* event rather than a richer log,
the fallback (`incident` + a new sub-`kind`, never a new top-level kind), and a
match-the-detail rule naming both production mislabels. `references/task-agent.md` gets
the equivalent for the agent side. `references/cycle-report.md` stays the full reference.

**Approach chosen** (user, this session): inline roster **plus** the no-invented-kinds
rule — over "read the reference at run open" (a resumed run loses it) and "fallback rule
only" (would have caught the four invented kinds but neither mislabel).

---

### F2 — Every timestamp was local time suffixed `Z` · **FIXED**

**Symptom.** All orchestrator- and agent-written timestamps in the run are local time
labelled `Z`, off by three hours from GitHub's. The run-dir stamp says `20260728-2309`;
GitHub says `02:09Z`. Per-task `spec→PR` is `not captured` for all 16 tasks because the
agent-written `specifiedAt`/`readyAt` are internally inconsistent — **several `readyAt`
postdate their own `mergedAt`**.

**Evidence.** Report §Tasks preamble; §Per-task ledger asterisk footnote; §Capture
failures; §Feedback → Missing ("A timestamp discipline").

**Traces to.** `references/cycle-report.md:29` specified the *format* ("ISO-8601, UTC")
but nothing said **where the value comes from**, so it came from the model's head.

**Class.** Missing capability.

**Changed.** A `date -u +%Y-%m-%dT%H:%M:%SZ` mandate in three places — `SKILL.md`
(Evidence capture, covering `ts` and every timestamped `run.json` field),
`references/task-agent.md` (covering `ts` and `specifiedAt`/`readyAt`/`lastCheckedAt`/
`approvedAt`), and `references/execution-state.md` (as a field note). GitHub's stamp is
declared authoritative where both exist, and mixing sources inside one duration is
forbidden. Plus a render-time guard in `references/cycle-report.md`: check the merged
stream for impossible orderings (events before `run.opened`, `readyAt` > `mergedAt`,
whole-hour disagreement with GitHub, negative durations) and render `not captured` —
explicitly **never** reconcile the two clocks by applying an offset, which would invent
the numbers the `not captured` rule exists to prevent.

**Approach chosen:** mandate at the source **and** the render-time check — over either
alone, because a run predating the fix would otherwise still render corrupt durations
silently.

---

### F3 — Merge-detection retired tasks with no agent, and cleanup double-spawned · **FIXED**

**Symptom.** Two opposite failures around the same moment. Cleanup was **missed on four
tasks** (T1, T4, T6, T12): marked `done` on merge-detection with no agent spawned, so
stale worktrees sat on disk and handoffs went uncaptured until a retroactive pass — found
only because the user asked about an unrelated task's retirement. `tasks/T12.json` still
reads `status: "open"` though #68 merged at 04:45:28Z. Meanwhile idle-gating was
**violated twice** (T2, T15): a cleanup agent spawned while the task's own agent was
still live, both writing the same task file concurrently.

**Evidence.** `run.json.cadenceDefects[0]`, `[3]`, `[7]`; `tasks/T12.json`; report
§What went wrong; §Feedback → Missing ("A merge-detection → cleanup rule").

**Traces to.** `SKILL.md:931` made the cleanup agent "recovery only" and named the normal
path (the monitor agent cleaning up in the tick that detects the merge) — but nothing
forbade the orchestrator taking the third path it actually took: marking `done` itself,
with no agent at all.

**Class.** Behavior defect.

**Changed.** A new spawn-table row (`open` whose PR just merged → **monitor agent**) and
a `A DETECTED MERGE IS A DELTA, NOT A CONCLUSION` block in `SKILL.md` step 2: the
orchestrator never writes a task file, never retires a task itself, never spawns cleanup
for a task with `agentInFlight = true`, never spawns cleanup alongside a monitor for the
same task, and treats a merged PR whose file still reads `open`/`fixing` after its agent
returned as an `incident` (`orchestrator.error`). Both production failures are quoted in
full, in both directions.

**Approach chosen:** the invariant-reinforcing shape (orchestrator never marks `done`) —
confirmed explicitly by the user, since it touches idle-gating and the single-writer
rule. Rejected: a post-hoc guard (leaves the double-write path open) and an
end-condition census (the four misses would still have happened, just been cleaned up
late).

---

### F4 — The orchestrator refused an empty branch as a base · **FIXED**

**Symptom.** T12's dispatch was held because T11's branch was deliberately empty,
serialising T12's Opus spec behind T11's entire implement + gate + self-review + PR.
The report is precise about the mechanism: *"an agent pushed an empty branch deliberately
to open the gate, and I overrode it as unsafe."*

**Evidence.** `run.json.cadenceDefects[6]`; §Feedback → Hurt ("'Branches published at
spec start' needs an orchestrator-side statement").

**Traces to.** The publish-at-spec-start rule (added 3.11.0) is written to the **task
agent**. Nothing on the orchestrator side said it must *accept* what that agent pushes.

**Class.** Behavior defect. **Partially** covered already: 3.12.0's sequencing gates
define base availability in terms of branches existing (`SKILL.md:908-915`), but never
state that an empty branch counts.

**Changed.** An `AN EMPTY BRANCH IS A VALID BASE` block on the dispatch bullet in
`SKILL.md` step 3: "exists" means the ref is on the remote — not that it has commits,
differs from its base, or has a PR. Withholding dispatch on emptiness is the same freeze
under a different name and is logged as an `incident` (`orchestrator.error`). What
governs *when* a dependent starts remains the sequencing gate.

---

### F5 — Stale observation passed to an agent as fact · **FIXED**

**Symptom.** The orchestrator told T16 the compose `db` container was running. True when
measured, false when used.

**Evidence.** `run.json.cadenceDefects[8]`; §Feedback → Missing ("Extend 'never the
derived number' to cover stale observations").

**Traces to.** The `STATE THE RULE, NEVER THE DERIVED NUMBER` invariant
(`SKILL.md:964-993`) names only counts, totals and arithmetic claims.

**Class.** Missing capability.

**Changed.** The invariant now covers observations explicitly, with the reasoning that
they fail identically — both were true when looked at and are asserted as true when the
agent acts hours later. It names the specific class the orchestrator must never assert:
anything it doesn't own and can't re-check for the agent (a service up, a port free, a
branch's contents, a file existing, a check's result, "the tree is clean"). Passed-on
observations are labelled `unverified` **with when they were observed**.

---

## Deliberately NOT changed

### D1 — "`tick` was never logged" and "the orchestrator never emitted `task.spawn`"

The report's headline capture complaint, and its §Feedback → Hurt item asks for *"a
one-line 'log a `tick` before re-arming' in the Re-arm step"*.

**Declined — already present, and has been since 3.11.0.** Verified:

```
git show cadence-v3.6.0:…/SKILL.md  → 'kind: "tick"' 0 · 'task.spawn' 0 · 'incident' 0
git show cadence-v3.11.0:…/SKILL.md → 'kind: "tick"' 1 · 'task.spawn' 1 · 'incident' 7
```

The run **opened under 3.6.0**, which had none of these rules; 3.11.0 added all three
(`SKILL.md:958-962` for spawns, `:1047-1055` for the tick, `:1119-1130` for incidents),
and the tick bullet already says "EVERY wake-up, quiet ones included" and spells out this
exact failure. The report is judging behavior the current version already forbids.

**What would settle it next cycle:** the Cost section. A run opened under ≥3.11.0 that
still renders `Ticks: not captured` means the rule is being read and ignored, which is a
different (and much more serious) finding than "the rule is missing". Until then there is
nothing to add — a second copy of an instruction is not enforcement.

### D2 — The cycle-label misdiagnosis (9 PRs unlabelled)

`run.json.cadenceDefects[2]`: the orchestrator recorded the cycle label as "vanishing
from the repo" when `gh pr edit --add-label` simply fails on colon-containing labels.

**Declined — the behavior was removed in 3.10.0.** Cadence no longer creates or applies
labels at all: `SKILL.md:878-880` ("Never create labels, and never label a PR"),
`references/pr-template.md:56`, `references/task-agent.md:418`, and
`references/execution-state.md:357` records `cycleLabel` as removed. There is no
labelling code path left to misdiagnose.

### D3 — T5's agent stayed alive and resumable after its task retired

`run.json.cadenceDefects[4]`, noticed only when the user pointed it out.

**Deferred — not enough evidence to act on.** The report doesn't say whether the agent
was still *doing* anything or merely still listed as resumable by the harness, and the
distinction decides whether this is a Cadence rule (agents must return, never idle) or a
harness artifact with no cost. Adding a rule against the second would be papering over a
non-symptom.

**What would settle it next cycle:** an `incident` at the moment it is noticed, recording
whether the agent was consuming tokens after its task hit `done`.

### D4 — The six repo-side follow-ups

Coverage reporting broken by `actions/download-artifact` extraction paths;
`.env.example` documenting a `pnpm --filter` command that can't work in a non-pnpm
workspace; `audit-append-only.real.spec.ts` needing `VOUCHER_DATABASE_URL` explicitly;
exit (e) sitting outside the gate behind `RUN_REAL_TESTS=1`; `migration:create`
regenerating the manual-exception partial-index drop.

**Not-the-plugin.** These are `pagana-catalog-apps` issues. Cadence found and reported
them correctly, which is the system working. No plugin change; they belong in that repo's
backlog.

---

## What the report confirms is working

Recorded because it is the comparison baseline for the next round:

- **The corrected join model (3.9.0/3.10.0).** 2 joins built, 2 retired, **0 stranded
  PRs, 0 recovery PRs**, against cycle 1's four stranded (`#38`/`#39`/`#40`/`#42`) and
  three recovery PRs. PR bases audited at three points; none ever targeted a join.
- **Version pinning (3.7.0).** The 3.11.0 upgrade landed mid-run and pinning stopped the
  orchestrator adopting a join topology the plan doc banned. The user then changed it
  explicitly — the escape hatch used exactly as designed.
- **"State the rule, never the derived number" (3.11.0), on first use.** T14 counted
  `REQUIRED_PATHS` itself (24), cross-checked the artifact (24), and found a task-id
  collision no brief mentioned. Agents refuted the orchestrator **four times** in this
  run (T1 on the flake premise, T9 on the `[3,2,10,6,1]` conflict, T15 on exit (b), T16
  on the stale pre-flight) — every one prevented a real error. Note that F5 above is the
  gap this rule left open, not a failure of the rule.
- **Zero re-drafts after ready**, and one correct draft-at-open (T14, CI pending) — the
  positive readiness test from 3.8.0 holding.

---

## Round summary

| Finding | Class | Outcome |
|---|---|---|
| F1 · invented/mislabelled event kinds | missing capability | fixed — `SKILL.md`, `task-agent.md` |
| F2 · timestamps are local time labelled `Z` | missing capability | fixed — `SKILL.md`, `task-agent.md`, `execution-state.md`, `cycle-report.md` |
| F3 · merge-detection retired tasks with no agent; cleanup double-spawned | behavior defect | fixed — `SKILL.md` |
| F4 · empty branch refused as a base | behavior defect | fixed — `SKILL.md` |
| F5 · stale observation passed as fact | missing capability | fixed — `SKILL.md` |
| D1 · `tick`/`task.spawn`/`incident` unlogged | — | declined (already fixed in 3.11.0; run opened under 3.6.0) |
| D2 · cycle-label misdiagnosis | — | declined (labelling removed in 3.10.0) |
| D3 · agent alive after retirement | — | deferred (needs one `incident` to classify) |
| D4 · six repo follow-ups | not-the-plugin | no change |

All five accepted findings are prose/rule edits to `cadence-executor` and its references;
no plan-doc, state-schema, or planner↔executor contract change. **MINOR** bump.
