# Execution state (per-run directory, split by owner)

Durable source of truth for an in-flight cycle run. Locate it at the start of every
scheduled wakeup; the owner of each file updates it after every action. It outlives
any single session.

**State is a directory, not one file** — so each task's agent can own and write its
own state without racing the others (parallel agents writing one shared JSON would
clobber each other):

```
.cadence/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/
  run.json                  ← ORCHESTRATOR-owned (the top loop is the only writer)
  tasks/<id>.json           ← TASK-AGENT-owned (that task's agent is the only writer)
  events/run.jsonl          ← ORCHESTRATOR-owned, append-only evidence log
  events/<id>.jsonl         ← TASK-AGENT-owned, append-only evidence log (one per task)
  report.md                 ← the cycle report, rendered from state + the event logs
```
- `<YYYYMMDD-HHMM>` = run start (`date +%Y%m%d-%H%M`); `<6char-hash>` =
  `openssl rand -hex 3`; `<slug>` = canonical slug from the plan metadata header.

**Ownership (strict):** the orchestrator reads everything but writes ONLY `run.json`
and never a `tasks/<id>.json`. Each task agent writes ONLY its own `tasks/<id>.json`.
A task's status source of truth is its `tasks/<id>.json` (absent file = `pending`);
`run.json` holds the roster + schedule and a cached last-reported status used only for
gating.

**Locate (every invocation):** glob `.cadence/cycles/*-<slug>-cycle/run.json`, pick the
dir whose `run.json.planPath` equals the current plan and isn't complete (newest
datetime prefix wins); only create a new dir if none matches. The name is NOT
reconstructable from the slug alone — always locate, never blindly re-create.

**Setup:** create `.cadence/` and ensure `.cadence/` is in the repo `.gitignore` (this state
must never be committed onto a feature branch).

## `run.json` (orchestrator-owned)

```json
{
  "slug": "matchmaking-followups",
  "runDir": ".cadence/cycles/20260625-1430-a1b2c3-matchmaking-followups-cycle",
  "runHash": "a1b2c3",
  "planPath": "docs/plans/proposed/20260625-1430-matchmaking-followups-ABC-1234.md",
  "createdAt": "2026-06-23T12:00:00Z",
  "ghLogin": "hugoseabra",
  "repo": "org/repo",
  "preflight": {
    "plugins": { "superpowers": "ok" },
    "graphify": "ok",
    "ghAuth": "ok",
    "mcp": { "Linear": "ok" },
    "passedAt": "2026-06-23T12:00:30Z"
  },
  "integrationBranch": "cadence/matchmaking-followups-integration",
  "integrationWorktree": ".claude/worktrees/cadence-matchmaking-followups-integration",
  "planPrNumber": 1200,
  "planPrUrl": "https://github.com/org/repo/pull/1200",
  "planPrStatus": "draft",
  "prTitlePattern": {
    "exemplar": "[ABC-1234] Add reply-correlation matcher",
    "note": "[<source-key>] <Title in sentence case>"
  },
  "issueTracker": {
    "tool": "Linear",
    "parentIssue": { "key": "ABC-400", "url": "https://linear.app/org/issue/ABC-400" },
    "workflowStates": ["Todo", "In Progress", "In Review", "Done", "Blocked"]
  },
  "monitorBackoff": { "baseSeconds": 180, "maxSeconds": 1800, "quietTicks": 2 },
  "nextWakeupAt": "2026-06-23T12:50:00Z",
  "prSnapshot": {
    "T1": {
      "updatedAt": "2026-06-23T12:20:11Z",
      "headRefOid": "9f1c2ab…",
      "baseRefName": "cadence/matchmaking-followups-integration",
      "mergedAt": null,
      "isDraft": true,
      "reviewDecision": "REVIEW_REQUIRED",
      "mergeable": "MERGEABLE",
      "mergeStateStatus": "CLEAN",
      "ciState": "SUCCESS",
      "unknownSince": null
    }
  },
  "refSnapshot": {
    "cadence/matchmaking-followups-integration": "4c7d1e0…",
    "cadence/matchmaking-followups-t1-add-reply-correlation-matcher": "9f1c2ab…"
  },
  "//refSnapshot": "Head OID of the integration branch and every task/join branch. A moved ref is a delta for EVERY PR based on it — this is how a merge the human just made wakes up its dependents, since their own PR fields don't change when their base moves.",
  "topology": {
    "0blockers": "integration", "1blocker": "that blocker's branch (stacked)",
    "2plus": "branch off a per-task join; PR TARGETS INTEGRATION, never the join",
    "source": "skill"
  },
  "//topology": "The base rule this run uses, PINNED at run open exactly like the merge policy. source: 'plan' when the plan doc states its own topology (the plan wins — the human read and approved it; record the skill rule that disagreed). On resume, if the installed skill's topology differs from this, KEEP THIS ONE, log an incident (kind: 'topology.skew') and surface it: an upgrade must never re-base a live cycle. This exists because a mid-run upgrade once swapped the base model against the plan and an explicit user instruction, stranding four tasks' merged work off integration.",
  "incidents": [
    { "at": "2026-05-04T11:20:00Z", "kind": "brief.false-premise", "taskId": "T4",
      "what": "brief asserted REQUIRED_PATHS goes 21 -> 23; path keys already existed, delta is 0",
      "caughtBy": "T4 spec agent, counted the live array", "action": "brief corrected; no code change" }
  ],
  "//incidents": "Cadence's OWN failures, recorded as they happen — wrong briefs, topology skew, task collisions, external merges that broke the cycle, anything the run got wrong. Every entry ALSO gets an `incident` event in events/run.jsonl (if you wrote a field, you owe an event). These are the most valuable lines in the run: they are what the cycle report's 'what went wrong' is built from, and the failure sections are never softened.",
  "policyVersion": "3.7.0",
  "//policyVersion": "The cadence version this run was OPENED under. Behaviour that the user was told about at run open — merge policy above all — is pinned to it. If the installed plugin is newer, note the skew ONCE in the turn summary and keep behaving as the run started; only the user can opt an in-flight run into new policy.",
  "approvalMergePolicy": "auto",
  "//approvalMergePolicy": "'auto' (default) = a task PR with an INTACT human approval is merged by its own agent once every guard passes (Approval-authorized auto-merge). 'off' = the human always clicks merge. The plan PR → main is NEVER auto-merged under either setting.",
  "mergeAuthorization": null,
  "//mergeAuthorization": "A SEPARATE, broader grant, independent of approvalMergePolicy. null = none. Set ONLY on an explicit user instruction, e.g. { scope: 'wave:1' | 'task:T3' | 'cycle', grantedBy: 'user', at: '<iso>', note: '<verbatim instruction>' }. Merge only within scope, only when green + not draft. Never covers the plan PR.",
  "attention": [
    { "kind": "decision", "taskId": "T2", "prNumber": 1207, "id": "D1",
      "question": "Purge or archive expired invites?",
      "url": "https://github.com/org/repo/pull/1207#issuecomment-99", "blocking": true },
    { "kind": "plan-pr-ready", "prNumber": 1200, "note": "merge #1200 into main to close the cycle" }
  ],
  "stalls": [
    { "id": "T4", "reason": "blocked-by-failed-dep", "sinceTick": "2026-06-23T13:10:00Z",
      "note": "T3 failed its gate; T4 cannot build its join" }
  ],
  "unresolvedDecisions": [],
  "reportPath": ".cadence/cycles/20260625-1430-a1b2c3-matchmaking-followups-cycle/report.md",
  "notified": ["decision:T2:D1", "plan-pr-ready"],
  "//notified": "Keys of attention items already pushed as a notification. Notify on the TRANSITION (item entering attention) and never again while it sits there — one notification per turn, max, and never for routine progress.",
  "modelPolicy": {
    "spec":    { "model": "opus",   "effort": "high" },
    "high":    { "model": "opus",   "effort": "medium" },
    "medium":  { "model": "sonnet" },
    "low":     { "model": "sonnet", "effort": "low" },
    "trivial": { "model": "sonnet", "effort": "low" },
    "monitor": { "model": "sonnet", "effort": "low" }
  },
  "roster": [
    { "id": "T1", "title": "Add reply-correlation matcher", "deps": [], "complexity": "medium",
      "lastStatus": "open", "agentInFlight": false, "agentKind": null, "agentStartedAt": null },
    { "id": "T2", "title": "Wire matcher into inbound pipeline", "deps": ["T1"], "complexity": "high",
      "lastStatus": "implementing", "agentInFlight": true, "agentKind": "implement",
      "agentStartedAt": "2026-06-23T12:40:00Z", "model": "opus" }
  ]
}
```

`complexity` is `null` until the task's **Spec phase** writes it; the orchestrator
then uses it to pick the Implement agent's model. `agentKind` ∈
`spec | implement | monitor | fix | cleanup`.

- `roster[].lastStatus` — orchestrator's cached copy of each task's status (from the
  agent's return summary), used for wave gating. Truth lives in `tasks/<id>.json`.
- `roster[].agentInFlight` — true while that task's agent is running (build/fix/
  cleanup, usually in the background). **The orchestrator skips any task with
  `agentInFlight: true`** — it spawns nothing and doesn't touch the PR (Idle-gating).
- `roster[].agentKind` — `spec | implement | monitor | fix | cleanup`, what the
  in-flight agent is doing (selects its model: spec → opus/high, implement → by
  `complexity`, monitor/cleanup → cheap).
- `roster[].agentStartedAt` — when it was spawned; if it exceeds the stale-lease
  window (~30m) with no completion, treat the agent as dead and recover.
- `roster[].complexity` — `high | medium | low | trivial`, **written by the task's Spec
  phase** (null until then); never assigned at plan/definition time. Selects the
  *implement* agent's `model`/effort via `modelPolicy`, and the Implement phase's
  pre-push review depth (`trivial` → skip `/code-review`; `low`/`medium` → `/code-review
  low`; `high` → `/code-review high`). `trivial` = no logic/behavior change (typo, lint,
  doc/comment only).
- `roster[].model` — the model the current in-flight agent was spawned with (for audit
  + escalation tracking).
- `tasks/<id>.json.branchPublishedAt` — when the task's agent pushed its branch to the
  remote. **This happens at spec start, before any code is written**, because a task's
  branch existing is what makes its dependents dispatchable. Publishing at PR-creation
  instead (the old behaviour) forced every dependent to wait out its blocker's build,
  gate, self-review and PR body — a chain then ran strictly end to end, which is a
  freeze wearing the name `waiting-for-blocker-branch`.
- `tasks/<id>.json.handoff` — **the only channel between agents that never share a
  context.** `{contract, delivered}`: `contract` is written at the end of the task's
  spec (the exact surfaces it will produce — names, signatures, routes, entities,
  migration names — plus its decisions and anything differing from the plan's brief);
  `delivered` is written when its PR opens (what actually shipped, drift from its own
  contract, and gotchas a consumer would otherwise rediscover). The orchestrator pastes
  every blocker's handoff verbatim into a dependent's brief, and **the handoff outranks
  the plan's brief** where they disagree — the brief predicted this code before it
  existed; the handoff describes what is there.
- `tasks/<id>.json.implementBlockedOn` — `{taskId, surface, expected, found}` when an
  implement agent found a consumed surface missing or differing from its blocker's
  `handoff.delivered`. Since implement only starts once every blocker's PR is open,
  this is a **contract mismatch** to surface (`incident`, `kind: "contract.mismatch"`),
  not a normal wait.
- `modelPolicy` — model/effort by phase: **spec/analysis → opus, high effort
  (always)**; **implement** → high→opus/medium, medium/low/trivial→sonnet; **monitor/
  fix/cleanup** → sonnet, low effort. Think on Opus, do routine work on Sonnet — don't run
  the whole cycle on Opus, and don't run analysis on a cheap model.
  **The tier only applies if the phase is actually a separate spawn**, so only
  `trivial` uses the fused fast path; a fused `low` would be implemented by the
  opus/high spec agent, which is how a cycle ends up running almost entirely on Opus
  while this table says otherwise.

## `tasks/<id>.json` (task-agent-owned)

```json
{
  "id": "T1",
  "title": "Add reply-correlation matcher",
  "deps": [],
  "status": "open",
  "complexity": "medium",
  "branch": "cadence/matchmaking-followups-t1-add-reply-correlation-matcher",
  "baseBranch": "cadence/matchmaking-followups-integration",
  "joinBranch": null,
  "joinedShas": null,
  "worktreePath": ".claude/worktrees/cadence-matchmaking-followups-t1-add-reply-correlation-matcher",
  "prNumber": 1203,
  "prUrl": "https://github.com/org/repo/pull/1203",
  "isDraft": true,
  "draftReason": "ci_red",
  "//draftReason": "Why it isn't reviewable: agent_in_flight | gate_red | ci_red | no_self_review | body_incomplete | blocking_decision | conflict | scope_polluted.",
  "readyAt": null,
  "baseHeadSha": "4c7d1e0…",
  "lastSyncedBaseSha": "4c7d1e0…",
  "filesChanged": 4,
  "scopeCheck": { "at": "2026-06-23T13:22:00Z", "verdict": "clean", "foreignPaths": [] },
  "lastCheckedAt": "2026-06-23T12:25:00Z",
  "pendingDecisions": [
    { "id": "D1", "question": "Purge or archive expired invites?",
      "options": ["A — archive (soft)", "B — purge (hard delete)"],
      "provisionalChoice": "A", "impact": "Irreversible for existing rows if B ships",
      "blocking": true, "status": "open",
      "commentUrl": "https://github.com/org/repo/pull/1203#issuecomment-99",
      "askedAt": "2026-06-23T12:24:00Z", "remindedAt": null,
      "answeredBy": null, "answeredAt": null, "resolvedInSha": null }
  ],
  "approvals": [
    { "login": "hugoseabra", "isHuman": true, "association": "OWNER",
      "state": "APPROVED", "approvedSha": "9f1c2ab…", "at": "2026-06-23T13:02:00Z" }
  ],
  "autoMerge": null,
  "//autoMerge": "Set only when this agent merged its own PR under an intact human approval: { authorizedBy, approvedSha, mergedSha, headDelta: 'identical'|'base-sync'|'trivial', guards: {…}, mergedAt, commentUrl }.",
  "answeredComments": [
    { "commentId": 882134, "threadId": "PRRT_kwDO…", "verdict": "agree",
      "replyUrl": "https://github.com/org/repo/pull/1203#discussion_r882140",
      "threadResolved": true }
  ],
  "reviewerFixRounds": { "<reviewer-login>": 2 },
  "decisionLog": [
    {
      "decision": "Reply matching strategy",
      "chosen": "Message-ID round-trip",
      "alternatives": ["recipient-email fallback only"],
      "why": "Highest precision; recommended by analysis",
      "howToRollback": "Comment 'use option B' → switch to email fallback"
    }
  ]
}
```

## Status values (a task's `status`, in `tasks/<id>.json`)
- `pending` — planned, not started (no file yet = pending). **Idle** → spawn a **spec** agent (opus/high).
- `specifying` — a spec agent is **in flight** (analysis + plan, deciding complexity). Skip.
  A spec agent that finds `complexity` = **`trivial`** (and only `trivial`) **fuses
  straight into the Implement phase in the same invocation** and ends at `open`;
  `low`/`medium`/`high` all stop at `specified`. Fusing means one invocation on one
  model, and spec always runs on opus/high — so anything fused is *built* on Opus
  whatever its complexity says. That is worth it for a typo and nothing more.
- `specified` — spec done, `complexity` and `handoff.contract` written (`low`/`medium`/
  `high` — only `trivial` fuses past this, and only when its blockers are already
  `open`), **idle** → spawn an **implement** agent at `modelPolicy[complexity]`
  **once every blocker's PR is open**. Until then the task sits at `specified` as
  `awaiting-blocker-pr` — legitimate sequence, not a stall: it clears on another
  agent's work, never on a human's merge.
- `implementing` — an implement agent is **in flight** (worktree+branch+code, pre-PR). Skip.
- `open` — PR created (base = the task's `baseBranch`: integration, or its single
  blocker's branch when stacked — **a 2+-blocker task branches off its join but its PR
  targets integration; the join is never a PR base**), **settled/idle**, awaiting
  human/CI → this is the ONLY status a Monitor pass runs on. `open` covers both draft
  and ready — the draft flag is `isDraft`, re-evaluated by the readiness checklist on
  every tick, never by asking the user.
- `fixing` — a fix agent is **in flight** pushing review/CI/conflict fixes; returns to
  `open`. Skip while in flight.
- `merged` — the task PR was merged **into its base** (integration, its blocker's
  branch when stacked, or its join branch) — by the human, by its own agent under an
  **intact human approval** (`approvalMergePolicy: "auto"`, all guards passed → see
  `autoMerge`), or under an explicit user merge authorization; cleanup next. The plan
  PR is never in this set by any route but a human's click.
- `done` — merged AND worktree/branch destroyed. Terminal (the task agent has died).
- `failed` — gate couldn't go green; PR left as draft with a blocker note for the human.

`specifying`/`implementing`/`fixing` imply `agentInFlight: true` (agent owns the task);
`pending`/`specified`/`open`/`merged` with `agentInFlight: false` are idle and get an
agent this tick.

`planPrStatus` (in `run.json`): `draft` (tasks in flight) → `ready` (all tasks merged,
un-drafted, awaiting human) → `merged` (human merged plan PR into `main` — run done).

## Field notes
- `runDir` / `runHash` — the located run directory and its 6-char token.
- `preflight` — BLOCKING gate (`plugins.superpowers` — REQUIRED, the run STOPS if the
  superpowers plugin is missing; `graphify` — `"ok"`/`"absent"`, optional accelerator,
  never a blocker; `ghAuth`; `mcp` per-server or `"none"`; `passedAt`). Execution
  can't start until `passedAt` is set; re-verified only before dispatching NEW
  spec/implement work — a pure monitor tick skips the pings.
- `monitorBackoff` — adaptive wakeup interval: `{baseSeconds, maxSeconds, quietTicks}`.
  A hot tick (delta / spawn / agent in flight) resets `quietTicks = 0` and sleeps
  `baseSeconds`; each fully quiet tick increments `quietTicks` and sleeps
  `min(baseSeconds × 2^quietTicks, maxSeconds)`. Defaults 180/1800. (A legacy
  `monitorIntervalSeconds` field, if present from an older run, is read as
  `baseSeconds`.)
- `prSnapshot` / `refSnapshot` — per-task change-detection baseline from the
  orchestrator's ONE batched read-only GraphQL call (`updatedAt`, `headRefOid`,
  `baseRefName`, `mergedAt`, `isDraft`, `reviewDecision`, `mergeable`,
  `mergeStateStatus`, `ciState`), **plus the head OID of `main`, the integration branch,
  and every task/join branch**. `main` is in there because the repo keeps moving while a
  cycle runs: an external merge can change the foundation the whole cycle is built on,
  and waiting for the plan PR to turn red is finding out too late — a moved `main` is a
  signal to merge it into integration and re-check, not a curiosity. Two rules that
  exist because conflicts were being missed: (1) **a moved ref is a delta for every PR
  based on it** — when the human merges, the dependents' own PR fields don't change, so
  refs are the only signal; (2) **`mergeable: UNKNOWN` is never "no news"** — GitHub
  computes mergeability lazily, so `UNKNOWN == UNKNOWN` must not read as quiet; stamp
  `unknownSince` and re-query next tick, and after ~3 tries determine it locally with a
  dry-run merge. A PR that is `DIRTY`/`BEHIND`/`CONFLICTING` is top priority and gets an
  agent every tick until clean. A monitor agent is spawned for an
  idle `open` task ONLY when a field differs from this baseline (or none exists).
  Re-baselined right after that task's agent completes, so the agent's own
  replies/pushes don't read as news next tick. Detection only — the orchestrator
  never triages or acts on PR content.
- `policyVersion` / `approvalMergePolicy` — **written explicitly at run open and then
  pinned.** `approvalMergePolicy` is never left absent, because absence is how a policy
  silently changes under a live run: an upgrade lands mid-flight, the re-entering
  `/cadence:ship` reads the *new* skill's default, and a run whose human was promised
  "merges are human-only" starts merging. So:
  - **A run.json with NO `approvalMergePolicy` predates the feature → treat it as
    `"off"` (human-only)**, write that value in, log a `policy.pinned` event, and say so
    once in the turn summary. Never infer the new default into an old run.
  - **If the installed version ≠ `policyVersion`**, note the skew once
    ("run opened under 3.2.1, plugin now 3.7.0 — merge policy stays as it started; say
    'adopt the new policy' to change it") and keep behaving as the run started.
  - **Only the user can change it mid-run**, explicitly. Then update both fields and log
    the authorization.

  This pinning covers the promises a human relies on: **merge policy, draft/readiness
  semantics, `main` safety, and the branch topology** (`topology`, above). Mechanical
  improvements (better conflict detection, cheaper ticks, richer reports) always apply
  immediately — they change nothing the user was told.
- `approvalMergePolicy` — `"auto"` (default **for new runs**) or `"off"`. Under `"auto"`, a task PR
  carrying an **intact human approval** is merged by its own agent once every guard
  passes (human approver, approval not superseded, head unchanged in substance since
  `approvedSha`, no unresolved decision, all comments answered, CI green, mergeable
  clean). The **plan PR → `main` is never auto-merged** under any setting.
- `reportPath` / `notified` — where the cycle report is written (announced at run open,
  echoed in every turn's footer, refreshed on demand by `/cadence:report`), and which
  `attention` items have already triggered a desktop notification so a pending item is
  never re-notified tick after tick.
- `attention` — orchestrator-owned, rebuilt every tick by reading the task files: the
  list of things that actually need the human, each with a link where they act (open
  decisions, parked review loops, failed tasks, PRs awaiting review >24h, the plan PR
  once ready). It is printed in every turn summary — a decision that lives only inside
  a PR comment thread is invisible, which is the failure this field exists to prevent.
- `stalls` — tasks that reported the same non-dispatchable reason for 3+ consecutive
  ticks (`{id, reason, sinceTick, note}`), from the **Flow audit**. A `waiting-for-merge`
  reason is a defect, not a stall: it must be converted (join / base-sync / re-target) the
  same tick.
- `unresolvedDecisions` — decisions that were still open when their PR was merged
  anyway (by a human): the question, the default that therefore shipped, and the PR
  link. Carried into the final run summary as follow-ups so nothing evaporates.
- **(removed) `cycleLabel`** — Cadence no longer creates or applies labels. The repo's
  label set belongs to its humans, and a run passing through for a few days does not
  write to it. Task↔PR identity is carried by the PR body's identity header, the plan
  PR's cycle map, and the naming rules. A `cycleLabel` left over in an older run's
  `run.json` is ignored.
- `events/run.jsonl` + `events/<id>.jsonl` — **append-only evidence logs**, one line per
  event (`{ts, actor, kind, detail, pr?, url?, sha?, model?}`), written by their single
  owner as things happen (the orchestrator's own log; one per task agent). They are the
  only durable record of a multi-day run — nothing is reconstructed at the end.
- `report.md` — the rendered **cycle report** (outcome, per-task table, what needed the
  human, what went wrong, flow health, cost, timeline, follow-ups, maintainer feedback).
  Written at the end condition and on demand (`/cadence:report`). Format, event kinds,
  and the no-fabrication rules: `cycle-report.md`.
- `integrationBranch` — stacked base; every task branches from it and PRs target it.
- `integrationWorktree` — the checkout for `integrationBranch`. Its creation is where
  the cycle's plan/design docs are swept off `main` and committed (step 1.3), and where
  plan-PR CI/comment fixes run. Removed at cleanup once the plan PR merges.
- `planPr*` — the plan PR (integration → `main`); the human merges it last.
- `prTitlePattern` — `{exemplar, note}` PR-title convention; the orchestrator refreshes
  it from a `renamedTitle` reported by a task agent so new PRs match the latest style.
- `nextWakeupAt` — when `ScheduleWakeup` re-enters. Monitoring is in-session only — no
  cron / external scheduled agent.
- `baseHeadSha` / `lastSyncedBaseSha` — the base's head when the branch was cut, and the
  base commit last merged in. Together they say whether this branch is current, and they
  are what makes a squash-merged parent detectable (the base advanced past
  `lastSyncedBaseSha` while this branch still carries the parent's original commits).
- `filesChanged` / `scopeCheck` — the size and honesty of the PR's diff:
  `git diff origin/<base>...HEAD --name-only` after every base sync. `verdict`:
  `clean` (only this task's files) or `polluted` (+ `foreignPaths`). A polluted diff is
  fixed — usually by re-resolving a squash-merge collision in the base's favour — **before**
  the PR is offered for review, and `filesChanged` is echoed in the PR body so a reviewer
  can spot a mismatch at a glance.
- `baseBranch` — a task's PR base, resolved from its blockers: the `integrationBranch`
  (no blockers), the **single blocker's branch** (stacked), or this task's
  **`joinBranch`** (2+ blockers). Never `main`. Base syncs and PR creation use this exact
  value; update it when a stacked base merges away or a join retires (then re-base onto
  integration and `gh pr edit --base` it).
- `joinBranch` / `joinedShas` — for a 2+-blocker task: the synthetic base
  `cadence/<slug>-t<id>-join` (integration + every blocker's branch merged in) and the
  blocker head SHAs it was built from. It exists so the task can start while its
  blockers are still open — it is infrastructure, never gets a PR, and is deleted once
  every blocker has merged into integration and the PR is re-targeted there. `null` for
  0/1-blocker tasks.
- `isDraft` / `draftReason` / `readyAt` — draft means **not reviewable yet**, not "not
  mergeable yet": an agent in flight, a red gate/CI, no self-review, an unwritten body,
  or an unanswered **blocking** open decision. Being stacked on an unmerged base is
  **not** a reason. The readiness checklist runs at PR-open and every monitor tick and
  the agent un-drafts itself the moment it passes — the user is never asked. `draftReason`
  records which item failed so the run summary can report the truth; `readyAt` stamps
  the un-draft.
  > **Every timestamped field in this file — `readyAt`, `specifiedAt`, `lastCheckedAt`,
  > `approvedAt`, `agentStartedAt`, `nextWakeupAt`, `unknownSince`, and any other — is
  > produced by `date -u +%Y-%m-%dT%H:%M:%SZ`, never composed from memory.** A `Z` on a
  > local time is false and corrupts every duration the cycle report computes; in
  > production it produced task files whose `readyAt` postdated their own `mergedAt`.
  > Where GitHub stamps the same moment, GitHub's value wins.
- `pendingDecisions` — questions that genuinely need a human (product/policy intent,
  ambiguous criteria, irreversible or security-relevant choices), each posted on the PR
  as an answerable numbered comment with options and a provisional default, pinned in
  the PR body, and mirrored into `run.json.attention`. `blocking: true` keeps the PR
  draft and disqualifies auto-merge until answered. `status`: `open | resolved`.
  Anything settleable on the merits is NOT here — it's a `decisionLog` line.
- `approvals` — per-reviewer approval ledger built from each tick's `reviews` payload:
  `{login, isHuman, association, state, approvedSha, at}`. `approvedSha` is the commit
  the reviewer approved — the anchor for the auto-merge staleness test. A later
  `CHANGES_REQUESTED` or a dismissal clears that reviewer's approval.
- `autoMerge` — set only when the task's own agent merged the PR under an intact human
  approval: who authorized it, `approvedSha` → `mergedSha`, the `headDelta` class
  (`identical` / `base-sync` / `trivial`), the guards that passed, and the URL of the
  authorization comment posted before merging (NO SILENT MERGES). An approved PR whose
  **base isn't landable** (a parent task's branch with a still-open PR, or a join
  branch) is not merged — it reports `approved_awaiting_base` and merges on the tick
  after the base lands. That's a waiting button, not a stalled task: its dependents are
  already building on its branch.
- `answeredComments` — `{commentId, threadId, verdict, replyUrl, threadResolved}` per
  handled comment. `replyUrl` is proof the reply posted (NO SILENT FIXES). For an
  agreed-and-fixed item, `threadResolved: true` records that the review thread was
  resolved (GraphQL `resolveReviewThread`) so an automated reviewer
  sees it as solved — a reply alone doesn't signal resolution. Declined/clarify items
  have `threadResolved: false` (left open for the human). Recorded only after the reply
  is confirmed, so the agent never double-replies, misses one, or marks a fix handled
  without a visible reply + resolution.
- `reviewerFixRounds` — per-reviewer count of completed fix→re-request rounds on this
  PR (REVIEW CONVERGENCE BOUND in `task-agent.md`). After **3** rounds with the same
  reviewer still returning new findings, the agent stops re-requesting that reviewer,
  posts one summary comment, and parks the remaining findings for the human — this is
  what bounds the fix → re-review churn loop with automated reviewers. Optional field;
  absent means zero rounds. Human comments and other reviewers are unaffected by the
  counter.

## Resume logic (each wakeup — top orchestrator)
0. **Locate** the run dir (glob `*-<slug>-cycle/run.json`, match `planPath`) — never
   create a duplicate. Read `run.json` + every `tasks/<id>.json`. **Then check the policy
   pin:** no `approvalMergePolicy` → the run predates the feature → set it to `"off"`
   (human-only) and say so once; `policyVersion` ≠ the installed version → note the skew
   once and keep the run's original behaviour. An upgrade never rewrites what this run's
   human was told at its start.
1. Re-verify the preflight gate **only if this tick will dispatch NEW spec/implement
   work** (auth/MCP can drop between sessions); a pure monitor tick skips the pings.
2. **Change detection:** ONE batched read-only GraphQL call over all the cycle's open
   PRs; diff each against `prSnapshot`; store the fresh values. No delta on an idle
   `open` task → spawn nothing for it (quiet) — unless its readiness/auto-merge guards
   or a base advance could now pass, or an open decision is due its one reminder.
2b. **Flow audit:** classify why every non-terminal task isn't advancing. Anything
   held because another PR hasn't merged is a **defect** — convert it this tick (build
   or refresh the join base, merge the base in, re-target) and dispatch it. Record 3-tick
   repeats in `stalls`. Rebuild `attention` from the task files.
3. **Spawn one per-task agent per IDLE active task with work** (its **base is
   available** — integration, its single blocker's branch when stacked, or all
   blockers' branches existing so a join can be built; NOT "blockers merged" —
   `agentInFlight: false`) in a single message, model by phase:
   `pending`→**spec** (opus/high; fuses into implement for `trivial` only),
   `specified`→**implement** (`modelPolicy[complexity]`), `open` **with a snapshot
   delta**→**monitor** (sonnet), `merged`→**cleanup** (sonnet; recovery only — the
   monitor agent normally cleans up in the tick that detects the merge); **skip any
   task with `agentInFlight: true` or status `specifying`/`implementing`/`fixing`**
   (its agent owns it — don't touch its PR). Set `agentInFlight` when spawning;
   spec/implement/fix run in the background. Every brief includes the path to
   `references/task-agent.md` (the agent reads it first) and `preflight.graphify`.
   Each agent resumes from its `tasks/<id>.json`, advances one step, writes its own
   file, returns a summary. The orchestrator does NO `gh` writes/fixing — its only
   GitHub access is the read-only change-detection call.
4. Clear `agentInFlight` for completed agents; update `run.json` from the summaries
   (cached `lastStatus`, `prTitlePattern` on a reported rename, plan-PR task lines
   added once on first open); **re-baseline `prSnapshot`** for each PR whose agent
   acted. Recover any agent past its stale lease.
5. **Dispatch (no merge gate):** any task whose **base becomes available** is active
   next tick — a stacked child the moment its blocker's branch is pushed, a
   multi-blocker task the moment all its blockers' branches exist. Flow; never freeze
   a task waiting on another's merge.
6. When all tasks are `done`/`failed` → un-draft the plan PR (`planPrStatus = ready`)
   on your own, put "merge #<n> into `main`" at the top of `attention`, and delegate
   plan-PR CI/comment fixes to an agent in the integration worktree. **Never merge the
   plan PR.**
6b. **Report honestly:** the turn summary lists each PR's true state (draft + why,
   awaiting review, awaiting decision, approved, auto-merged, merged) and prints
   `attention`. Never "all ready for review" while any PR is a draft.
6c. **Append this tick's events** to `events/run.jsonl` (one `tick` line with spawns /
   deltas / quiet / interval, plus any orchestrator-level event: preflight, stalls,
   flow conversions, escalations, plan-PR transitions).
7. If any task is active OR `planPrStatus ≠ merged` → **ScheduleWakeup again**
   (mandatory — turn-end invariant) at the adaptive interval from `monitorBackoff`:
   hot tick → reset `quietTicks`, sleep `baseSeconds`; fully quiet tick → increment
   `quietTicks`, sleep `min(baseSeconds × 2^quietTicks, maxSeconds)`. Only once the
   plan PR is `merged` into `main`: remove the integration worktree/branch, **render
   `report.md`** from the event logs, write the summary with its path, no wakeup.
