# Cycle report — evidence of what went well and what didn't

**Purpose.** When a cycle ends (or at any point the human asks), Cadence produces
**one self-contained file** — `<runDir>/report.md` — that a human can read, paste into a
chat, or hand to a maintainer as the evidence record of the run: what shipped, what
needed a human, what went wrong, and what it cost.

The report is **evidence, not narration.** Every claim in it points at something
checkable — a PR number, a comment URL, a commit SHA, a timestamp, a state field. If a
fact wasn't captured while it happened, it does **not** get invented at render time; it
is reported as `not captured`.

---

## 1. Capture as you go (a run spans days and sessions — memory won't survive it)

Both levels append to **their own** append-only JSONL log, preserving the same
single-writer rule as the state files:

- **orchestrator** → `events/run.jsonl`
- **each task agent** → `events/<id>.jsonl`

One event per line, appended (never rewritten), small enough to append atomically:

```json
{"ts":"2026-06-23T12:41:07Z","actor":"T2","kind":"pr.opened","detail":"opened as draft","pr":1207,"url":"https://github.com/org/repo/pull/1207"}
```

Fields: `ts` (ISO-8601, UTC), `actor` (`orchestrator` or the task id), `kind`, `detail`
(one plain sentence), plus whichever of `pr`, `url`, `sha`, `model`, `ms`, `count` apply.

**`ts` is produced by `date -u +%Y-%m-%dT%H:%M:%SZ`, never composed from memory** — by
the orchestrator and by every task agent alike. A model-written clock read is a guess
that looks authoritative; a `Z` on a local time is simply false. See the timestamp rules
in `SKILL.md` (Evidence capture) and `task-agent.md`.

**Event kinds to log** — the left column is what the report is built from, so log the
event the moment it happens, not at the end:

> ### THE STATE FILES ARE CURRENT VALUES; THE EVENT LOG IS THE LEDGER
> `run.json` and `tasks/<id>.json` are overwritten in place — they answer "what is true
> now", never "what happened". So **every mutation of task state must emit its event in
> the same breath**, or the history is gone for good:
> - a task's `status` changes → `state.changed` (`from`, `to`, what caused it);
> - the orchestrator observes a PR snapshot field change → `snapshot.delta` (which
>   fields, `old` → `new`) — this is the ledger of everything that happened *to* a PR:
>   reviews appearing, CI flipping, draft state, mergeability, head SHA;
> - a base ref moves → `base.moved` (`ref`, `old` → `new` oid, which tasks depend on it);
> - a PR is re-targeted, synced, or scope-checked → `pr.retargeted` / `basesync` /
>   `scope.checked`.
>
> Rule of thumb: **if you wrote a field, you owe an event.** A report that can't show
> when a PR became conflicted and when it was fixed is missing the exact evidence the
> reader needs.
>
> Two capture failures seen in production, both of which gut the report:
> **(1) no `tick` events at all** — the run could not say how many times it woke, how
> many ticks were quiet, or whether the backoff worked, so the whole Cost section
> rendered `not captured`; **(2) an event log thinner than the state files** — three
> orchestrator errors, a cross-session collision, an open risk and an answered decision
> existed only in `run.json`, so the timeline was missing precisely the run's most
> important moments. Log the `tick` every wake-up, and log an `incident` the moment
> anything goes wrong, **especially when Cadence itself is what went wrong.**

| Kind | Logged when | Feeds the report's… |
|---|---|---|
| `policy.pinned` | the run's merge policy is pinned or a version skew is detected (`policyVersion`, installed version, resulting policy) | header, **what went wrong** |
| `run.preflight` | preflight passes/fails (record which checks, and `graphify` ok/absent) | header, cost |
| `state.changed` | any task `status` transition (`from` → `to`, trigger) | per-task ledger, timeline |
| `snapshot.delta` | the orchestrator sees PR fields change (`fields`, `old`→`new`) | per-task ledger, review/CI health |
| `base.moved` | a base ref's head OID changes (which dependents it affects) | flow health, conflict latency |
| `basesync` | a base is merged into a task branch (`method`: update-branch \| local merge) | flow health |
| `conflict` / `conflict.resolved` | a sync conflicts; how it was resolved (base-favoured hunks) | **what went wrong** |
| `pr.retargeted` | the PR's base changes — by you, or silently by GitHub after a branch delete | flow health |
| `scope.checked` / `scope.polluted` | the post-sync diff-vs-base check (`filesChanged`, foreign paths) | **what went wrong** |
| `branch.published` | a task pushes its branch at spec start (empty), making its dependents dispatchable | flow health, timeline |
| `handoff.written` | a task records `handoff.contract` (end of spec) or `handoff.delivered` (PR opened) — the knowledge its dependents build on | flow health, timeline |
| `task.spawn` | **any** agent is spawned — spec/implement/monitor/fix/cleanup *and* every ad-hoc repair, verification or housekeeping agent (`agentKind`, `model`, effort) | cost, timeline |
| `task.complexity` | the spec phase decides `complexity` (+ whether it fused) | per-task, cost |
| `join.built` / `join.refreshed` / `join.retired` | a join branch is created, re-merged, or retired | flow health |
| `pr.opened` | the PR is created (draft or not) | per-task, timeline |
| `pr.ready` / `pr.redraft` | un-drafted, or sent back to draft (with the reason) | readiness health |
| `decision.raised` / `decision.reminded` / `decision.answered` / `decision.shipped_unresolved` | an open decision's lifecycle (`id`, question, `blocking`, comment URL, who answered, latency) | **what needed you** |
| `review.received` | a review lands (`reviewer`, `state`, human/bot) | review health |
| `review.round` | a fix→re-request round completes for a reviewer | review health |
| `review.parked` | the 3-round convergence bound trips for a reviewer | **what went wrong** |
| `comment.answered` | a comment is judged + replied (verdict, `replyUrl`) | review health |
| `ci.red` / `ci.green` | CI flips (which checks failed) | **what went wrong** |
| `fix.pushed` | a fix commit is pushed (`sha`, what it fixed) | timeline |
| `guard.blocked` | an auto-merge guard held the merge back (which guard, why) | merge health |
| `automerge` | the agent merged under an approval (`authorizedBy`, `approvedSha`→`mergedSha`, `headDelta`) | outcome |
| `merged` | the PR merged (by whom) | outcome |
| `incident` | **Cadence got something wrong** (`kind`: `brief.false-premise` \| `topology.skew` \| `task.collision` \| `orchestrator.error` \| `external.merge` \| …), what caught it, what was done | **what went wrong** |
| `stall` | the flow audit records a 3-tick stall (reason) | **what went wrong** |
| `flow.converted` | a `waiting-for-merge` was converted (how) | flow health |
| `escalation` | a task was re-spawned a model tier up, or a stale lease was recovered | **what went wrong**, cost |
| `gate.failed` | the lint/format/tests gate failed (attempt number) | **what went wrong** |
| `task.failed` | a task is given up on, with the blocker | outcome |
| `tick` | **every** wake-up, quiet ones included: `spawns` (by kind), `deltas`, `quiet`, `quietTicks`, `sleptSeconds` | cost |

Keep `detail` to one sentence. Never log secrets, tokens, or file contents.

---

## 2. Render the report

**Where it lands, and how the user learns about it:** `<runDir>/report.md`, inside the
target repo's gitignored `.cadence/`. The path is announced in full at run open, carried
as a one-line footer on **every** turn, printed whenever the file is written or
refreshed, and led with in the final summary. When the cycle finishes, that's also the
one moment worth a desktop notification (`PushNotification`, if available):
`"cycle <slug> complete: N PRs merged · report at <path>"`. The user should never have to
ask where the report is, or discover it only by looking.

**When:**
- **Always** at the end condition (plan PR merged, or the run abandoned) — write
  `report.md` *before* the final summary, and print its path.
- **On demand** — `/cadence:report` (or the user asking "how did the cycle go") renders
  the current state of an in-flight run, clearly marked `IN FLIGHT`.

**How:** read `run.json`, every `tasks/<id>.json`, and every `events/*.jsonl`, merge the
events by `ts`, and fill the template below. Compute durations from the events; leave a
field as `not captured` rather than estimating one.

> ### SANITY-CHECK THE CLOCKS BEFORE YOU RENDER A SINGLE DURATION
> Locally-written timestamps can be wrong (see the `date -u` rule above), and a wrong
> clock that agrees with itself renders as a confident, false number. So before computing
> anything, check the merged event stream for **impossible orderings** and refuse to
> render across them:
>
> - an event that precedes the run's own `run.opened`;
> - a task's `specifiedAt` / `readyAt` that **postdates** a later-stage stamp
>   (`readyAt` > `mergedAt`, `specifiedAt` > `readyAt`, an event after `merged`);
> - a locally-written `ts` that disagrees with the **GitHub** timestamp for the same
>   moment by a whole number of hours — that is a timezone bug, not latency;
> - any negative duration.
>
> **A duration derived from a suspect pair renders `not captured`, never a number** —
> and the report says why, once, in *What went wrong*: "orchestrator/agent timestamps are
> local time labelled `Z` (offset −Nh); GitHub-sourced timestamps are authoritative and
> unmarked; locally-derived durations are omitted." Mark any timestamp you kept from a
> suspect source with an asterisk and footnote it. **Never quietly reconcile the two
> clocks by applying an offset** — you would be inventing the very numbers the `not
> captured` rule exists to prevent. Where both exist, GitHub wins.

**Tone:** flat and factual. This report is how the human learns what to fix — in their
process *and* in this plugin — so **it must be as specific about Cadence's own failures
as about anything else.** A report with an empty "what went wrong" section on a run that
had three re-drafts and a parked review loop is a broken report.

---

## 3. Template — `<runDir>/report.md`

```markdown
# Cycle report — <slug>

**Status:** COMPLETE | IN FLIGHT | ABANDONED · **Plan:** <planPath> · **Repo:** <owner/repo>
**Started:** <createdAt> · **Ended:** <ts or "—"> · **Wall clock:** <Xd Yh>
**Plan PR:** #<n> (<state>) · **Run dir:** <runDir>
**Opened under:** cadence <policyVersion> · **Merge policy:** <auto | human-only>
<if the installed version differs, say so here — the run kept its original policy>

## Outcome

| | |
|---|---|
| Tasks | <n> planned · <n> merged · <n> failed · <n> still open |
| Task PRs | <n> merged (<n> auto-merged on your approval, <n> merged by you) |
| Needed you | <n> decisions · <n> review requests · <n> parked review loops |
| Cost | <n> agent spawns (<n> opus / <n> sonnet) · <n> ticks (<n> quiet) |

<One paragraph, plain language: did the cycle deliver what the plan said, and what got
in the way. No spin — if half the tasks stalled, that's the sentence.>

## Tasks

| Task | What it does | PR | Complexity | Outcome | Spec→PR | PR→merged | Reviews | Decisions |
|---|---|---|---|---|---|---|---|---|
| T1 | Add the reply-correlation matcher | #1206 | medium | merged (auto, approved by @you) | 22m | 3h 10m | 2 rounds w/ bot | — |
| T2 | Wire it into the inbound pipeline | #1207 | high | merged (by you) | 41m | 1d 2h | 1 human | D1 (4h to answer) |

## Per-task ledger

> One block per task: everything that happened to it and to its PR, in order, from the
> event log. This is the "what changed, and when" record — state files only ever show
> the final value, so this is the only place the history exists.

<details><summary><b>T2 · Wire the matcher into the inbound pipeline · #1207</b></summary>

| When | What changed | Detail | Evidence |
|---|---|---|---|
| <ts> | status `pending → specifying` | spec agent spawned (opus/high) | — |
| <ts> | complexity = `high` | 4 files in touch set | — |
| <ts> | status `implementing → open` | PR opened as draft | #1207 |
| <ts> | `isDraft true → false` | readiness checklist passed | <comment url> |
| <ts> | base `t1-branch → integration` | GitHub re-targeted it — #1206 merged, branch deleted | #1207 |
| <ts> | `mergeStateStatus CLEAN → DIRTY` | base moved: #1206 squash-merged into integration | <base.moved ts> |
| <ts> | conflict resolved, synced | merged base in; 3 hunks taken from base (stale copies of #1206) | <sha>, <comment url> |
| <ts> | scope check | 4 files changed — matches touch set | — |
| <ts> | `reviewDecision → APPROVED` | @you | <review url> |
| <ts> | status `open → merged` | auto-merged on your approval | <sha> |

**Conflict latency:** base moved <ts> → detected <ts> (<n>m) → clean again <ts> (<n>m).
</details>

## What needed you

> Every item a human had to resolve, with how long it waited. This is the section that
> shows whether the run was actually autonomous.

| # | What | Where | Raised | Answered | Waited |
|---|---|---|---|---|---|
| D1 | Purge or archive expired invites? | #1207 <comment url> | <ts> | <ts> — "archive" | 4h 12m |
| — | Approval on #1206 | #1206 | <ts> | <ts> | 3h 01m |

**Unresolved at merge:** <none | D<n> — the default that shipped + how to change it>

## What went wrong

> Defects, dead ends, and churn — Cadence's own included. Each with evidence.

- **<what happened>** — <one sentence of consequence>. Evidence: <PR/comment URL, SHA, ts>.
- *(examples of what belongs here: a task stalled N ticks and why · a
  `waiting-for-merge` that had to be converted · a review loop parked at 3 rounds ·
  a PR re-drafted after being marked ready · CI red more than once for the same cause ·
  a gate that never went green · a model escalation · a stale agent lease recovered ·
  an auto-merge guard that blocked a merge and why · a decision that shipped
  unresolved · a rebase that changed behavior.)*

**Nothing went wrong** is a valid finding — but only write it if the event logs really
are clean, and say which checks you ran to conclude that.

## What went well

- **<what worked>** — <why it mattered>. Evidence: <link/ts>.
- *(fused fast paths that saved a spawn · a join that let a blocked task start early ·
  a review answered and resolved in one round · a clean auto-merge.)*

## Flow health

| Signal | Count | Notes |
|---|---|---|
| Joins built / refreshed / retired | <n>/<n>/<n> | <which tasks> |
| Blocker dispatched → dependent dispatched | <median> · <worst> | how long a dependent waited for its blocker's **branch** to appear. Branches publish at spec start, so this should be about one tick; large values mean they are being published late, and that is the serialization this row exists to catch |
| Contract mismatches | <n> | dependents that found a consumed surface missing or different from their blocker's `handoff.delivered` — each is a handoff that was written too vaguely, and worth naming |
| Base syncs · conflicts resolved | <n> · <n> | <how many were squash-merge collisions> |
| **Conflict latency** | median <n>m · worst <n>m | base moved → detected → clean again |
| Scope-check failures | <n> | <PRs whose diff showed foreign files, and why> |
| Silent re-targets by GitHub | <n> | <PRs GitHub re-based when a parent branch was deleted> |
| `waiting-for-merge` conversions | <n> | should be > 0 only if the audit caught something |
| Stalls (3+ ticks) | <n> | <reasons> |
| Re-drafts after ready | <n> | <reasons — each one is a readiness miss> |

## Cost

| | |
|---|---|
| Agent spawns | spec <n> · implement <n> · monitor <n> · fix <n> · cleanup <n> · other <n> |
| By model | opus <n> · sonnet <n> · escalations <n> |
| Ticks | <n> total · <n> quiet · longest backoff <n>s |
| Fused fast paths | <n> `trivial` tasks finished in their spec invocation (spawns saved) |

## Timeline

<Compact chronological digest of the merged event logs — one line per event:
`<ts> · <actor> · <kind> — <detail> <url>`. Collapse repetitive ticks into
"<n> quiet ticks" runs so the timeline stays readable.>

## Follow-ups

- [ ] <anything left open: unresolved decisions, parked reviewer findings, failed tasks,
      deferred scope from the plan's NOT DOING list that this run touched.>

---

### Feedback for the Cadence maintainer

> Paste this section (or the whole file) to the plugin author. It is the run's evidence
> of where the tooling itself helped or hurt.

**Cadence version:** <plugin version> · **Cycle:** <slug> · **Repo shape:** <language/CI in one line>

**Worked:** <1–3 bullets with evidence links.>
**Hurt:** <1–3 bullets with evidence links — be specific: which rule fired, what it did,
what the human had to do about it.>
**Missing:** <what the human had to do by hand that Cadence should have handled.>
```

---

## 4. Rules

- **Never fabricate a metric.** No event → `not captured`. Durations are computed from
  logged timestamps only.
- **Never hide a failure to make the run look good.** The "what went wrong" section is
  the point of the file; an empty one on a messy run makes the whole report worthless.
- **Anchor every claim.** PR number, comment URL, commit SHA, or timestamp — a reader
  must be able to verify any line without asking.
- **The report is local.** It lives in `<runDir>` under `.cadence/`, which is
  gitignored — it is never committed to a branch or pasted into a PR body wholesale.
- **Regenerating is cheap and idempotent.** Re-rendering overwrites `report.md` from the
  same logs; it never edits the logs themselves.
