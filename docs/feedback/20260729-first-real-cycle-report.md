# Feedback round — 2026-07-29 · the first real cycle report

**Evidence:** `report.md` for cycle `voucher-phase2-c1-exceptions-foundations` in
`pagana-platform/pagana-catalog-apps` — 18 tasks, 25 task PRs (#31–#55), plan PR #30
merged at `cb630fd`, run dir
`.cadence/cycles/20260728-1626-208231-voucher-phase2-c1-exceptions-foundations-cycle`.
The run **opened on 3.2.1 and was reloaded to 3.6.0 mid-run at ~tick 4**.

Plus two direct observations from the user during triage:
- every task appeared to run on Opus;
- Cadence was creating and applying a PR label.

**Anchoring note.** The run predates `3.10.0`, so every finding was re-checked against
the *current* files before it was accepted. Several of the report's complaints were
already fixed; two of its biggest were **not**, and one was still live in a form the
report's own author never saw.

---

## Findings and what happened to each

### 1. F-A — the join-branch contradiction · **behavior defect · FIXED**

**Symptom (report §"What went wrong" 1):** #38, #39, #40 and #42 merged into join
branches instead of integration. A join has no PR, so nothing carried four tasks' work
onto integration; recovery took four steps and three PRs (#41, #45, #46). Left
unresolved, T19's exit gate would have run without `libs/storage` and the plan PR would
have merged without it.

**What the report blamed:** an asymmetry between 3.6.0's auto-merge guard 9 (which
refuses a join-based PR) and the readiness checklist (which had no equivalent item), so
in a human-merge-only run the only reachable control was silent.

**What was actually still wrong in 3.10.0.** The invariant had already been rewritten to
"branch off the join, target integration" — in the overview, the retirement rule, guard 9
and `CLAUDE.md`. But **two of the three places an agent actually reads before creating a
PR still said the opposite**:

- `cadence-executor/SKILL.md:527` — "The task branch is cut from the join; **the PR
  targets the join**", three paragraphs below the invariant that forbids it and directly
  contradicting its own retirement paragraph ("the PR targeted integration all along").
- `references/task-agent.md:295` — the `gh pr create` step: "that base is the integration
  branch (no blockers), the single blocker's branch (stacked), or **your join branch
  (2+ blockers)**". This is the line the PR-opening agent reads.
- `references/execution-state.md` — the `open` status note repeated it a third time.

So the 3.10.0 fix had landed on the *description* of the topology and not on the
*instructions*. This would have reproduced F7 exactly.

**Changed:** all three sites now say the PR targets integration, with the
`pagana-catalog-apps` incident quoted at the point of use and an explicit "if you have
just built a join, re-read this line before `gh pr create`."

### 2. F-B — "CI has no failing check" · **behavior defect · FIXED**

**Symptom:** two premature un-drafts. T5 (#36) un-drafted after observing "no CI checks
report on this branch" — the checks had not been created yet. T7 (#39) un-drafted with
two checks still `QUEUED`; the agent reported it honestly rather than claiming green.
T15 hit the same cause as a `guard.blocked`.

The report's diagnosis was exactly right and is quoted in the fix: *"the 3.6.0 checklist
item is 'CI has no failing check' and queued is not failing — but the intent is a
terminal-green gate."* The wording was still verbatim in 3.10.0 (`SKILL.md:980` and the
agent's own checklist).

**Changed:** readiness item 2 is now a positive test — rollup on the **current head
SHA**, check set **non-empty**, every check **terminal** (`SUCCESS`/`NEUTRAL`/`SKIPPED`),
none failed. Anything else → stay draft with `draftReason: "ci_pending"`, which is not a
stall (the tick that sees green un-drafts it). The only exemption is a repo *confirmed*
to have no CI, recorded as `ciExpected: false` with the reason. Mirrored in
`task-agent.md`, `CLAUDE.md` and the README.

### 3. F-C — a mid-run upgrade re-based a live cycle · **missing capability · FIXED**

**Symptom:** the plan's §7 and an explicit user instruction both said "0 or 2+ blockers →
integration". The 3.2.1 → 3.6.0 reload substituted the join model silently. Quoted from
`recoveryPlan.rootCause`: *"When the plugin upgraded 3.2.1 -> 3.6.0 mid-run I substituted
its join-branch model."*

Policy pinning existed in 3.10.0 — but its pinned set was **merge policy, draft/readiness
semantics, `main` safety**. Branch topology was not in it, which is precisely the promise
that got broken.

**Changed:**
- **Topology joins the pinned promises**, in `SKILL.md`, `execution-state.md` and
  `CLAUDE.md`.
- `run.json.topology` is written at run open (the 0/1/2+ base rule + `source`).
- **The plan doc wins** over the skill's default when they disagree — the human read and
  approved that document — recorded as `topology.source = "plan"` with the conflicting
  skill rule kept alongside, and stated in one line at run open.
- On resume, a topology that differs from the pinned one keeps **the pinned one**, logs
  an `incident` (`kind: "topology.skew"`) and surfaces it in `attention`.

### 4. F-D — briefs asserting derived numbers · **missing capability · FIXED**

**Symptom (report §"What went wrong" 2):** three `ORCHESTRATOR-ERROR` incidents, all
caught by the receiving agent rather than the orchestrator:

- **(a)** a false premise relayed to T10 (`apps/voucher-api/CLAUDE.md` is generated) —
  taken from another agent's report and passed on unverified. T10's agent verified,
  found it false, **declined**, and recorded `coordinatorRequestDeclined`.
- **(b)** T16's touch set "10 → 11 files" — a double count; the actual total was 10.
- **(c)** T18's `REQUIRED_PATHS` "21 → 23" — confusing OpenAPI **path keys** with
  **operations**. Both keys already existed, so the real delta was **zero**. The report:
  *"An implement agent trusting my number would have padded the array until it read 23
  and shipped a FALSE arithmetic docblock."*

The run's own recorded lesson — *"I should state the RULE and have the agent count, not
assert the number"* — is the fix, and it had no home in the plugin.

**Changed:** a **STATE THE RULE, NEVER THE DERIVED NUMBER** invariant in the dispatch
section, with the three incidents quoted and ❌/✅ phrasings. A number that must be passed
on is labelled **unverified** with its source. An agent that finds a brief false
**declines and reports it** — explicitly correct behaviour, recorded as an `incident`
(`kind: "brief.false-premise"`).

### 5. F-E / F-H — the capture gaps · **missing capability · FIXED**

**Symptom:** the report's Cost section is almost entirely `not captured` — no `tick`
events exist at all, and 13 `task.spawn` events undercount a run that spawned repair,
verification, housekeeping and monitor agents. Separately, the event ledger ended up
**thinner than the state files**: the orchestrator errors, the T18 collision, R3 and D1
exist only in `run.json`. The report calls this out itself: *"the state files carry more
history than the ledger, which is the opposite of the intended design."*

The `tick` kind and the "if you wrote a field, you owe an event" rule both already
existed in 3.10.0 — declared in `cycle-report.md`, never *instructed* anywhere in the
orchestrator's actual turn sequence. A rule in a reference nothing executes is a rule
that doesn't run.

**Changed:**
- the `tick` event is now emitted **as part of the re-arm step**, every wake-up including
  quiet ones, with `spawns`/`deltas`/`quiet`/`quietTicks`/`sleptSeconds`;
- **every** spawn logs `task.spawn`, ad-hoc repair/verification/housekeeping agents
  included ("there is no such thing as an unlogged spawn"); the report's cost table gains
  an `other` column;
- a new **`incident`** event kind, with `run.json.incidents` in the schema, and the rule
  that anything non-routine written to `run.json` gets its event **first**;
- both production capture failures are written into `cycle-report.md` as the reason.

### 6. F-F — the monitor poll loop · **missing capability · FIXED**

**Symptom:** T2's sonnet monitor did its base-sync correctly, then entered a wait-for-CI
loop instead of returning a verdict. Four re-notifications, no action, **~80k subagent
tokens per pass**, killed with `TaskStop`. The report's own prevention note is the fix.

**Changed:** a **NEVER WAIT INSIDE A TICK** block at the head of the Monitor pass — no
`sleep`, no polling, no retry-until-green, no re-running the GraphQL call hoping for a
different answer. CI still running *is* the answer: return `prState = ci_pending`
(added to the state vocabulary). Waiting belongs to the orchestrator's backoff, where it
is free.

### 7. F-I — the stale task file · **behavior defect · FIXED**

**Symptom:** T16's task file stalled at the spec phase ("nothing implemented, nothing
merged") while #50 had long since merged — its implement agent never wrote its own task
file on completion. Reconciled by hand against `gh`.

**Changed:** a task-file write contract at the top of the agent playbook — never return,
successfully or otherwise, without writing status and everything learned this invocation
plus the matching events. *"A step you completed but did not write down did not happen."*

### 8. F-G — two orchestrators on one task · **missing capability · FIXED (light)**

**Symptom:** a T18 collision with a concurrent session, caught only because the other
session's agent could not reach its own peer and relayed here by chance. The branch
existed **locally only** at `56b7944`, unpushed, no PR — nothing protecting the work.

**Chosen scope:** the cheap guard, not a lease. Before spawning a `pending` task's first
agent, check the remote: branch exists, or a PR matching `prTitlePattern` for that task
id is open → **do not spawn**, record `possible-concurrent-session` in `attention`, log
`incident` (`kind: "task.collision"`), let the user decide who owns it. Adopting,
double-PRing and force-pushing are all named as wrong.

### 9. F-L — `main` moving under a live cycle · **missing capability · FIXED**

**Symptom (report §4, F1–F6):** PR #37 (`chore/shared-docker-infra`) merged into `main`
mid-cycle by a human, not part of the cycle. It renamed the tables T1's **already-merged**
migration indexed (F1, HIGH — `migration:up` fails on any fresh DB), made T3's
`applyGuardSql` throw at runtime while its structural unit test still passed (F2, HIGH),
and stale-ended the snapshot and 15 doc lines (F3/F4). None of it was visible to the
cycle's gates; it surfaced as the plan PR going `CONFLICTING`.

`refSnapshot` watched integration and the task/join branches — **not `main`**.

**Changed:** `main`'s head OID is in `refSnapshot` and the batched GraphQL call. A moved
`main` triggers a prompt merge into integration (merge, never rebase — the plan PR is
open), a scope check against both parents, and any breakage it reveals is a first-class
finding: `attention` + `incident` + the report. "Waiting for the plan PR to turn red is
finding out too late."

### 10. Model policy — everything running on Opus · **behavior defect · FIXED**

**Raised by the user during triage**, and corroborated by the report: T5 and T7 are both
`low` complexity, and their logged spawns are `claude-opus-5[1m]`.

**Root cause, and it was by design.** Spec is unconditionally Opus/high, and the **fused
fast path** had the spec agent implement `trivial` *and* `low` tasks in the same
invocation. One invocation is one model — so every `low` task was **built on Opus**,
while `medium` built on Sonnet and `high` on Opus/medium. The cost curve was inverted:
the cheapest tasks got the most expensive model, and the policy table said otherwise.

**Changed:** **only `trivial` fuses.** `low` now ends at `specified` and gets its own
sonnet/low implement agent. The extra spawn is the price of the tier actually applying;
`trivial` keeps the fast path because there the spawn genuinely costs more than the
model. Synced across `SKILL.md`, `task-agent.md`, `execution-state.md`, `ship.md`,
`CLAUDE.md` and the README, each stating *why* rather than just the new rule.

### 11. PR labels · **behavior defect (by the user's call) · REMOVED**

**Raised by the user during triage** ("Now PRs are being created with a label.
Terrible!"), with supporting evidence in the report: `gh pr edit --add-label` failed with
"not found" on T6, T8, T13 and T17 — four identical `guard.blocked` events for a repo
that simply had no such label.

This was identity surface (2) of four. The user chose to drop it entirely rather than
make it conditional.

**Changed:** Cadence no longer creates a label, applies a label, or asks for one.
Identity now rests on three surfaces — the PR body's identity header, the plan PR's cycle
map, and the "never a task id without its PR number and a description" rule. `cycleLabel`
is removed from the schema (a leftover value in an older run's `run.json` is ignored) and
from the run-open announcement, the PR template, the report header, `CLAUDE.md` and the
README.

### 12. The real freeze — branches published only at PR-creation · **behavior defect · FIXED**

**Raised by the user after the round above shipped:** *"the behavior of waiting PR merges
to continue creating others still remains, freezing the work completely."*

**Corroborated live, mid-cycle**, from a running `voucher-phase2-c2-statements-dashboards`
turn summary:

> Merged (9): T1–T9 · Implementing: T10 · Pending (6): T11–T16
> T11 is convergence-blocked on T10, so there's genuinely nothing else to dispatch —
> **the tail is serial by design.**

Six tasks idle behind one, and the orchestrator concluding that was correct.

**Root cause, and why every anti-freeze rule missed it.** The merge-wait rules were all
present and being followed — `waiting-for-merge` is a defect, the flow audit runs every
tick, dependencies are expressed by base. But a task's branch is pushed in exactly **one**
place: `task-agent.md`, **Implement step 5, at PR creation**. Dispatch requires a
blocker's branch to *exist on the remote* (`SKILL.md:531`, `:560`, `:803`). So a dependent
waited for its blocker's spec **and** TDD build **and** lint/format/tests **and**
complexity-scaled self-review **and** PR body before it could start — and the flow audit
classified that as `waiting-for-blocker-branch` → **✅ legitimate**.

It was a merge-wait in everything but name, blessed by the very table written to abolish
merge-waits. That is why fixes 1–11 above didn't touch it.

**The honest trade-off, put to the user.** Literal zero-wait isn't free: dependencies
exist because a task *calls* its blocker's code, so a dependent that starts implementing
immediately has a red gate until the producer lands. Three models were offered; the user
chose the middle one — **publish early and push continuously**.

**Changed:**
- **The branch is pushed at spec start**, empty, before any code — the moment the worktree
  exists. An empty branch is identical to its base, has no PR, and is not reviewed; it
  costs a ref. This makes a dependent dispatchable **one tick after its blocker is
  dispatched** rather than after it finishes, so every task's expensive Opus spec phase
  runs in parallel across the cycle.
- **Commits are pushed as they're written**, not saved for PR-creation, and always
  immediately after landing a surface a dependent consumes. *The gate protects the PR, not
  the branch* — a branch with no PR is not under review.
- **Dependents verify their producers instead of waiting.** An implement agent merges its
  base in and checks the surfaces it consumes are actually present. If one isn't, it
  builds everything else, pushes, records `implementBlockedOn`, and returns to
  `specified`.
- **The re-spawn costs nothing and needed no new machinery.** `refSnapshot` already
  watches every task branch, so the blocker's next push moves its ref — and a moved ref is
  already a delta that fans out to every dependent in the same tick. The push *is* the
  signal; nothing polls.
- `waiting-for-blocker-branch` is now a **defect** once the blocker has been dispatched
  (its agent failed to publish). Never force-push: a dependent may already be based on
  what was pushed.
- New: `branch.published` / `blocked.producer` events, `branchPublishedAt` /
  `implementBlockedOn` state, and two flow-health rows — *blocker dispatched → dependent
  dispatched* and *`blocked.producer` returns* — so the next report shows directly whether
  branches are being published on time.

**Note on the in-flight cycle.** The `c2-statements-dashboards` run is open right now.
Branch-publication timing is **mechanical**, not a pinned promise (the pinned set is merge
policy, draft/readiness, `main` safety, topology — and the base *derivation* is unchanged
here), so a running orchestrator picks this up on its next wake **if** it reads these
files rather than a separately-installed copy. Worth checking before assuming the tail
un-freezes on its own.

---

## Deliberately NOT changed

- **The transport flake (R1).** Three signatures, one cause: `supertest@6` re-binding to a
  fresh ephemeral port per request and closing it asynchronously. Diagnosed and fixed in
  the repo (`app.listen(0)`, 3-of-10 failures → 0-of-15). **Not the plugin** — no Cadence
  rule would have found it faster than the agents did, and the run's own handling of it
  (refusing to call two unexplained failures "flake") is the behaviour we want.
- **#53 vs T4 — the `lint:check` mechanism collision (R3, HIGH, still open).** Two PRs
  fixing the same defect two ways, one from a concurrent session. The report classifies it
  correctly as *"a human design choice, not an orchestrator call."* Cadence should surface
  it — which it did — not resolve it. No rule added.
- **Auto-delete-on-merge silently re-targeting PRs** (#35 when T2's branch was deleted;
  T18's base before its PR existed). Already handled: `pr.retargeted` is logged and base
  moves are first-class deltas. The report records it as a standing repo hazard, which is
  the right place for it.
- **A run lease for concurrency (the stronger F-G option).** One occurrence, in a
  situation the user created by running two sessions. The pre-dispatch check covers the
  observed failure; a lease adds state, staleness rules and an abandoned-lease recovery
  path for a second data point we don't have yet.
- **Tokens and dollars in the report.** A subagent's spend isn't observable to the
  orchestrator, so any figure would be invented — and inventing metrics is the one thing
  the report format forbids. `task.spawn` + `tick` now give a real spawn and tick census;
  cost stays `not captured` rather than estimated. The report's single token figure
  (~80k/pass on the T2 loop) came from prose, and that is honest about what it is.
- **Escalating the report's "3.6.0's command text re-litigates settled policy" into a
  rewrite of `/cadence:ship`'s description.** The description documents the default, which
  is correct for a *new* run. Instead, `ship.md` now states that on an already-open run
  `run.json` wins over the command text and the pinned policy is not re-litigated on
  resume — the actual complaint (being asked twice about a settled human-only policy),
  without making the command lie about its own default.

---

## What to look for in the next report

Each fix has a field or section that will show whether it worked:

| Fix | Where it shows |
|---|---|
| F-A join topology | `pr.retargeted` events and any `cycle-repair` PR — both should be **absent**, and every task PR's base should read `integration` or a blocker's branch |
| F-B readiness | "Re-drafts after ready" and premature un-drafts in **Flow health** — the failure mode is a PR ready with `ci_pending` in its history |
| F-C topology pin | `run.json.topology` present with its `source`; a `topology.skew` incident if a version changes mid-run |
| F-D brief honesty | `incidents[]` with `kind: "brief.false-premise"` — **finding some is success**, it means they're being caught *and* recorded |
| F-E/F-H capture | the **Cost** section having real numbers at all; `tick` count vs. quiet count; `incidents` present in `events/run.jsonl`, not just `run.json` |
| F-F no-poll | no monitor agent burning tokens without returning; `ci_pending` appearing as a normal `prState` |
| F-I task files | no reconciliation-against-`gh` paragraph in **what went wrong** |
| F-G collisions | `task.collision` incidents, if the user runs concurrent sessions again |
| F-L `main` watch | `external.merge` incidents raised *before* the plan PR goes `CONFLICTING` |
| Model policy | the **Cost** table's spawn census by model — `low` tasks should show a sonnet implement spawn |
| Labels | no `guard.blocked` events for labels; nothing created in the repo's label set |

---

## A note on the report itself

This is the first report the format has produced from a real multi-day run, and it did
its job: it surfaced the orchestrator's own three errors, named a defect in its own
recovery diagnosis (`ROOT_CAUSE_CORRECTED`), corrected a false claim it had already made
to the user about who opened #52, and refused to soften "what went wrong." Two of the
findings above (F-A, F-B) are fixes to rules the report identified *precisely*, in the
report's own words, which is the loop working as designed.

The one structural weakness it exposed is that **the ledger under-captures relative to
state** — which is now a rule rather than an intention.
