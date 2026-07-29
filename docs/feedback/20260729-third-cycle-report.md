# Feedback round — third real cycle (`voucher-phase2-c3-sync-notifications`)

**Source:** `<runDir>/report.md` from `pagana-platform/pagana-catalog-apps`, cycle
`voucher-phase2-c3-sync-notifications`, rendered while **IN FLIGHT** (plan PR
[#73](https://github.com/pagana-platform/pagana-catalog-apps/pull/73) open, un-drafted,
awaiting the human's merge).
**Run:** 16 tasks planned · 16 merged (#74–#89) · 0 failed · 6h 36m from plan PR open to
last task merge · integration head `c32b9ae`.
**Cadence version:** opened under **3.12.0**.
**Baseline for this round:** 3.12.0 + the two `Unreleased` commits below.
**Released as:** 3.13.0 (this round + the two before it).

## Which fixes were live during this run

This determined half the triage, so it was verified rather than assumed:

| Commit | Pushed | Live? |
|---|---|---|
| `d423fdc` — closed event vocabulary, `date -u`, merge→cleanup, empty base, stale observations | 06:42:58Z | **Yes**, 34 min before run open |
| `554c607` — c3 plan doc: join topology restored | 07:14:43Z | **Yes**, ~2 min before |
| `0a9c090` — planner/executor `topology.stale` guard | **07:16:28Z** | **No** — the same second the run opened |

`d423fdc` demonstrably worked. Every anomaly this run is an `incident` carrying a real
sub-kind — `orchestrator.lost-dispatch`, `topology.skew`, `preflight.stale`,
`brief.false-premise`, `scope.widened`, `orchestrator.false-gotcha`, `spec.corrected` —
where cycle 2 invented `cadence.defect`. 50 ticks were logged, against zero in cycle 2.
The plan-doc fix is vindicated in the report's own words: *"Cycle 1's 'NO join branches'
conclusion was the wrong lesson and the live plan was right to supersede it."*

`0a9c090` missing the run by seconds is why finding F8 below exists in the form it does:
the guard would have caught the stale pin (the stale text contains a retired shape), but
the **root cause** — reading a copy from context instead of the file on disk — was still
uncovered, and is fixed here.

---

## Findings

### F8 — `agentInFlight` was trusted as truth; ~6h of critical path lost · **FIXED**

**Symptom.** T3 was written to state as `lastStatus: implementing` / `agentInFlight: true`
at 07:40:08Z and **its implement `Agent` call was never issued**. Idle-gating then skipped
it on every tick for ~6h; its branch sat empty at the base commit `554c607`; **T7 and T13
were blocked the entire time**. Caught at 09:29:53Z only by a lease check prompted by T3
outlasting every sibling implement. The same run failed in the **opposite** direction: T7
and T8's spec agents were spawned but never recorded.

**Evidence.** `run.json.incidents[orchestrator.lost-dispatch]` (with `relatedPriorIncident`
for the mirror case); `run.jsonl` `incident`; #84 opened 09:39:32Z with its dependents
#86/#87 forced to 10:09/10:13.

**Traces to.** `SKILL.md:465` — the stale-lease guard existed, but as a passive bullet in a
"why re-spawn each tick" aside, phrased as a ~30-minute stopwatch. A stopwatch neither
spares a slow live agent nor catches a dead one quickly; this one sat six hours.

**Class.** Behavior defect (rule present, too weak and in the wrong place).

**Changed.** An `agentInFlight IS A CLAIM, NOT A FACT` block in the per-tick path: before
deciding what to spawn, every in-flight task must show evidence an agent touched it —
branch head moved, task-file mtime advanced, or PR changed. A full tick with none of the
three → `incident` (`orchestrator.lost-dispatch`), reconcile from git/PR, clear the flag.
The flag is now written **from** the spawn (after the `Agent` call is issued, in the same
breath as `task.spawn`) rather than in anticipation of it, the reverse direction is
reconciled too, and a lost dispatch lands in `run.json.stalls` — the report noted it was
logged twice as an incident and **never once as a stall**, so the ledger a human consults
for delays omitted the run's biggest one.

**Approach chosen** (user): reality-checked lease every tick, over "confirm the spawn then
write the flag" alone (which prevents the lost dispatch but leaves a mid-flight death
undetected).

---

### F9 — The run pinned its topology from a stale in-context copy of the plan · **FIXED**

**Symptom.** At 04:12:31Z `run.jsonl` logged `policy.pinned` with `"NO join branches"`,
quoting cycle-plan §7.3. That quote came from the **pre-`554c607` draft held in the
orchestrator's context**. The live document — commit `554c607`, authored 07:14:43Z, ~2
minutes before run open — mandates per-task joins for T7/T8/T10/T13/T15/T16 and says in
terms: *"Do not re-import cycle 1's NO join branches conclusion from an older plan doc; it
is superseded."*

**Evidence.** `run.json.incidents[topology.skew]`; three independent refusals — T1
(`incident` "DOC/STATE DIVERGENCE"), T2 (`incident` "plan.divergence", which also caught
that the brief quoted T2's §4 text "verbatim" from the stale revision), T4 (credited in
`caughtBy`). Re-pinned 07:35:00Z; T1 and T2 re-based onto T14.

**Traces to.** `SKILL.md:384` said to write the topology at run open but never said
**where to read it from**. `0a9c090` (not live) narrows what may be pinned, but assumes
the right document was read.

**Class.** Behavior defect.

**Changed.** Pin time now does `git rev-parse HEAD`, re-reads the plan **file**, records
`topology.planSha` + `topology.planReadAt`, and quotes the §-number with the plan's exact
live text in the `policy.pinned` event. The note in the skill is blunt about why: three
agents doing the orchestrator's reading for it is not a safety net.

---

### F10 — Preflight assumed the quality loop instead of probing it · **FIXED**

**Symptom.** Two capabilities the run depends on were never checked, and both failed
silently.

- **Zero reviews on seventeen PRs.** Every PR #73–#89 has `reviewDecision: ""`,
  `reviews: []`, `reviewRequests: []`. T3 found the mechanism:
  `gh pr edit 84 --add-reviewer hugo-pagana` **returns exit 0 while `reviewRequests` stays
  empty** — GitHub silently refuses to add a PR's own author, and the cycle's `ghLogin`,
  the authenticated CLI account and the PR author are all one login. The review loop is
  **structurally inert in this repo configuration**, and the run described those PRs as
  awaiting review throughout.
- **`/code-review` carries `disable-model-invocation`** in this environment and cannot be
  called by an agent. Rediscovered independently by **thirteen tasks** (T1, T2, T4, T5, T6,
  T8, T9, T10, T11, T12, T13, T15, T16), each substituting a delegated subagent.

**Evidence.** `gh pr view` on all 17 PRs; T3's `guard.blocked` at log 07:36; the thirteen
task files each recording the substitution.

**Class.** Missing capability.

**Changed.** Preflight step 6 probes both and records the fallback: `preflight.reviewers =
"ok" | "self-only"` (under `self-only`, a PR is never again called *awaiting review* — it
awaits the human's **merge**, which is a different and honest sentence), and
`preflight.codeReview = "ok" | "unavailable"` relayed in every brief. `--add-reviewer`'s
exit code is explicitly not to be trusted: verify `reviewRequests` after the call.

**Also fixed alongside it — `preflight.stale`.** Preflight recorded `graphify: ok` from the
planning worktree, but `graphify-out/` is **gitignored**, so fresh task worktrees had no
graph; caught independently by T2 and T3's spec agents while briefs kept promising graph
queries. Any preflight fact resting on untracked state now records **where** it holds
(`"main-checkout-only"`), never a bare `ok`.

**Approach chosen** (user): probe both and record the fallback once, over detecting only
the self-review case or documenting without probing.

---

### F11 — Gotchas were broadcast as facts without evidence · **FIXED**

**Symptom.** G9 was recorded at 08:53:13Z with `relayTo: "every remaining implement
brief"`, claiming `--testPathPattern` is ignored by the integration jest project. **It is
the positional path that is ignored there.** T15 tested both forms under `pnpm jest`,
`pnpm exec jest` and the raw binary and inverted it. The failure mode is the dangerous
kind — **an all-green trap**: a "single-file" run that quietly executes the whole project
reads as a pass. Compounded by G11 (T7): `jest -t` takes a *regex*, so
`-t 'notification delivery (C3 T7)'` matched nothing, ran **zero** tests, and reported
all-skipped — also green.

**Evidence.** `run.json.incidents[orchestrator.false-gotcha]`; `T15.jsonl` `incident` at
log 15:01.

**Traces to.** The STATE-THE-RULE invariant governs *briefs*. A harvested `knownGotchas`
registry relayed into every brief is a second channel with no evidence bar at all — and it
multiplies a wrong entry by the number of agents it reaches.

**Class.** Missing capability.

**Changed.** A `RELAYED "GOTCHA" MEETS THE SAME EVIDENCE BAR AS A PR CLAIM` block: every
relayed entry carries the command that proved it or a `file:line`; no provenance → keep it,
label `unverified`, name who reported it. Two standing rules fall out and go into the
briefs — **prefer a gotcha that names a command over one that states a conclusion**, and
**an all-green result is not evidence a test ran** (check suite/test counts and the tail
line `Ran all test suites matching /…/` vs a bare `Ran all test suites.`). A gotcha later
found wrong is an `incident` **and** a correction pushed to every agent that received it.

> This is the same family as the four brief false-premises the run also caught — `Role.flags`
> asserted "verified in-tree" when the property is `Role.permissions`; three `buildEnvelope`
> snippets that would have published an event the app's own hop-2 consumer cannot parse,
> invisible to typecheck because the payload column is `unknown`; "two `CacheModule`
> clients" (false — Nest dedupes identical dynamic-module registrations); and an inverted
> `hasMore` that would have returned **an unbounded result set flagged `hasMore: false`**
> on the single most common call. All were declined by the agent that checked. The existing
> invariant covers those; F11 closes the relay channel it did not.

---

### F12 — State froze mid-truth when the orchestrator ran out of context · **FIXED**

**Symptom.** `run.json.prSnapshot` still says the plan PR `isDraft: true` (GitHub: false)
and shows all sixteen task PRs as open. `tasks/T3, T7, T10, T13, T15, T16.json` still read
`status: "open"` with `mergedAt: null` although all six merged. `run.json.exitVerdict.caveat`
still says *"no task PR has merged"*. A `tick.note` records the cause: *"handoff block
written to run.json; orchestrator context nearly exhausted"*. A T8 cleanup agent had
already exhausted its context and been respawned; two further agents are suspected and
had to be reported `not captured` because no event kind exists for it.

**Evidence.** `run.json`; six task files; `run.jsonl` tick notes; report items 7 and 14.

**Traces to.** Nothing forced a closing reconcile, and context exhaustion had no countable
representation.

**Class.** Missing capability.

**Changed.** A `FLUSH STATE BEFORE YOU RUN OUT OF ROOM` block on the end condition:
reconcile-and-flush is a step (every merged PR must have a task file saying so — spawned to
its monitor agent, since the orchestrator still never writes task files; `prSnapshot`
re-baselined; `attention`/`stalls`/`exitVerdict` rebuilt; falsified caveats rewritten), run
also on any tick where context is getting long. **Render the report early rather than
perfectly.** And an agent stopping early is now `incident` / `agent.exhausted` — countable
instead of free text.

---

### F13 — Ledger integrity: a malformed line silently dropped 11 events · **FIXED**

**Symptom.** Three separate defects in the run's own ledger.

- **One `events/run.jsonl` line holds two concatenated JSON objects** (a truncated tick note
  `"…run 100"` immediately followed by a complete rewrite of the same tick). `jq` exits 5
  there and **silently drops the 11 events that follow**. Recovered only with a tolerant
  parser: 72 objects, 1 unparseable line.
- **Placeholder timestamps.** T6's entire spec block and T15's spec/join/branch block are
  stamped `00:00:00Z`–`00:00:07Z`. Plus implement events stamped *before* their own spec
  events (T8, T10, T16), and `run.jsonl`'s final ticks reading 20:40–23:40 for merges
  GitHub timestamps at 12:48–13:44.
- **The two spawn ledgers disagree threefold.** `run.jsonl` carries **16** explicit
  `task.spawn` events; its own `tick.spawns` counters sum to **51**; task logs add 19 more.
  No reconciled total is capturable, so the Cost section cannot be rendered honestly.

**Evidence.** Report items 11, 12 and the Cost section.

**Class.** Missing capability. (The `date -u` mandate from `d423fdc` was live and did not
prevent placeholder stamps — it said where a *real* timestamp comes from, not that a fake
one is forbidden.)

**Changed.** A `ONE EVENT = ONE LINE, WRITTEN WHOLE` block: compose the object, append it
with its newline in one operation, never edit or rewrite an appended line — a correction is
a **new** line. Placeholder timestamps banned outright (omit the field; a missing `ts`
renders `not captured`, a fake one renders a wrong number). And the spawn ledgers must
reconcile — emit the event first, count the tick from events actually written, and if they
still disagree at render time **report both numbers and the discrepancy**, never the
flattering one.

---

## Deliberately NOT changed

### D5 — Two mislabelled event kinds

`T1.jsonl` logs `kind: "gate.failed"` with `detail: "none — full gate green on first pass"`;
`T9.jsonl` logs `ci.red` for a rollup that was merely **queued**. A naive count overstates
failures by one each.

**Declined — the rule landed in `d423fdc` and was live for this run.** "Make the `kind`
match the `detail`" is stated in both `SKILL.md` and `task-agent.md`, with both production
mislabels quoted as examples. Two slips across a run of 459-plus events, in a cycle that
otherwise used the vocabulary correctly throughout, is compliance with noise — not a
missing rule. Restating it a third time is the duplication D1 was declined for.

**What would settle it next cycle:** the count. If cycle 4 shows mislabels *increasing*
despite the rule, the problem is the rule's placement, not its existence.

### D6 — Cleanup incomplete (worktrees and join branches left behind)

Six task worktrees and five join worktrees on disk, twelve local branches, and **five join
branches still on origin** (`t7`, `t10`, `t13`, `t15`, `t16`; only `t8-join` was retired).

**Deferred — it is a symptom of F12, not an independent finding.** The cleanup writes were
due at exactly the point the orchestrator's context ran down. The reconcile-and-flush step
covers the state files; whether it also reliably retires joins is a question for cycle 4,
and adding a second cleanup rule before knowing that would be guessing.

**What would settle it next cycle:** `join.retired` count vs `join.built` count. It should
be 1:1.

### D7 — `(f.6)` reported UNPROVEN

The exit audit could not demonstrate one clause: it needs a PR whose changed paths are
`apps/voucher-api/**` only **and** whose branch carries T12's commit as an ancestor, and no
PR in the cycle can have both properties.

**Not-the-plugin, and the behavior was correct.** T16 chose to report it UNPROVEN on both
#73 and #89 rather than open a throwaway PR to manufacture a green checkmark. That is
exactly what the "never soften the failure sections" rule asks for. Recorded here as a
positive, not a finding.

### D8 — The remaining follow-ups

`VOUCHER_TENANT` as a sixth boot variable, `pnpm db:up` not working from a worktree,
`git branch -d` false-negatives against local `main`, re-checking T12's fallback against
#90, back-filling `(f.6)`.

**Not-the-plugin.** All `pagana-catalog-apps` items; several are already recorded as
durable gotchas (G13, G14) for cycle 4's plan.

---

## What this run confirms is working

The comparison baseline for cycle 4:

- **`d423fdc`'s vocabulary rule.** Every anomaly logged as `incident` with a real
  sub-kind, against cycle 2's invented `cadence.defect`. **50 ticks logged** vs zero.
- **The join model, decisively.** Six joins built, six refreshed, **zero conflicts across
  all twelve merge operations**, including T16's **fifteen-branch** join. The report's own
  verdict: *"The join model is vindicated in this repo… Cycle 1's 'NO join branches'
  conclusion was the wrong lesson."*
- **Stacking unwound itself.** #77/#78 (off T14) and #81 (off T1) were silently re-targeted
  onto integration by GitHub when their parents' branches were deleted, staying
  MERGEABLE/CLEAN with **no manual intervention** — confirmed twice independently.
- **Readiness discipline: 16/16 PRs held draft until CI was terminal-and-green, zero
  re-drafts**, every one logging `draftReason: ci_pending` rather than un-drafting on an
  empty check set.
- **NO SILENT FIXES held.** T5's CI red posted with root cause, ruled-out hypotheses and
  fix SHA *before* being recorded as handled.
- **Verify-the-brief is the highest-value rule in the skill.** Seven false orchestrator
  claims caught, two of which would have shipped silent runtime defects invisible to
  typecheck and to CI.
- **A `git diff` clean-tree gate beats file-read verification** — it caught two live
  mutation leaks in a PR whose stated scope was tests only. Worth promoting to a default
  for mutation-testing tasks in a future round; not done here, since one run is one data
  point.

---

## Round summary

| Finding | Class | Outcome |
|---|---|---|
| F8 · `agentInFlight` trusted as truth; ~6h lost | behavior defect | fixed — `SKILL.md` |
| F9 · topology pinned from a stale in-context plan copy | behavior defect | fixed — `SKILL.md` |
| F10 · preflight assumed the review loop and `/code-review` | missing capability | fixed — `SKILL.md` |
| F11 · gotchas broadcast as facts without evidence | missing capability | fixed — `SKILL.md` |
| F12 · state froze when the orchestrator ran out of context | missing capability | fixed — `SKILL.md` |
| F13 · malformed JSONL line dropped 11 events; placeholder stamps; spawn ledgers 16 vs 51 | missing capability | fixed — `SKILL.md`, `cycle-report.md` |
| D5 · two mislabelled event kinds | — | declined (rule live and largely obeyed; 2 slips) |
| D6 · cleanup incomplete | — | deferred (symptom of F12; watch `join.retired` next cycle) |
| D7 · `(f.6)` UNPROVEN | not-the-plugin | no change — correct behavior |
| D8 · repo follow-ups | not-the-plugin | no change |

All six accepted findings are rule edits to `cadence-executor` and its references; no
plan-doc, state-schema or planner↔executor contract change. **MINOR** — shipped as
**3.13.0** together with the two preceding rounds.
