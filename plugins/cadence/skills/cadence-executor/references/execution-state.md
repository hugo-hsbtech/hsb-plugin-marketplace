# Execution state (per-run directory, split by owner)

Durable source of truth for an in-flight cycle run. Locate it at the start of every
scheduled wakeup; the owner of each file updates it after every action. It outlives
any single session.

**State is a directory, not one file** — so each task's agent can own and write its
own state without racing the others (parallel agents writing one shared JSON would
clobber each other):

```
.cadence/cycles/<YYYYMMDD-HHMM>-<6char-hash>-<slug>-cycle/
  run.json            ← ORCHESTRATOR-owned (the top loop is the only writer)
  tasks/<id>.json     ← TASK-AGENT-owned (that task's agent is the only writer)
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
  "preflight": { "ghAuth": "ok", "mcp": { "Linear": "ok" }, "passedAt": "2026-06-23T12:00:30Z" },
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
  "monitorIntervalSeconds": 180,
  "nextWakeupAt": "2026-06-23T12:50:00Z",
  "mergeAuthorization": null,
  "//mergeAuthorization": "null = default (never merge; human merges). Set ONLY on an explicit user grant, e.g. { scope: 'wave:1' | 'task:T3' | 'cycle', grantedBy: 'user', at: '<iso>', note: '<verbatim instruction>' }. Merge only within scope, only when green + not draft.",
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
- `modelPolicy` — model/effort by phase: **spec/analysis → opus, high effort
  (always)**; **implement** → high→opus/medium, medium/low/trivial→sonnet; **monitor/
  fix/cleanup** → sonnet, low effort. Think on Opus, do routine work on Sonnet — don't run
  the whole cycle on Opus, and don't run analysis on a cheap model.

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
  "worktreePath": ".claude/worktrees/cadence-matchmaking-followups-t1-add-reply-correlation-matcher",
  "prNumber": 1203,
  "prUrl": "https://github.com/org/repo/pull/1203",
  "isDraft": true,
  "lastCheckedAt": "2026-06-23T12:25:00Z",
  "answeredComments": [
    { "commentId": 882134, "threadId": "PRRT_kwDO…", "verdict": "agree",
      "replyUrl": "https://github.com/org/repo/pull/1203#discussion_r882140",
      "threadResolved": true }
  ],
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
- `specified` — spec done, `complexity` written, **idle** → spawn an **implement** agent at `modelPolicy[complexity]`.
- `implementing` — an implement agent is **in flight** (worktree+branch+code, pre-PR). Skip.
- `open` — PR created (base = the task's `baseBranch`: integration, or its single
  blocker's branch when stacked), **settled/idle**, awaiting human/CI → this is the
  ONLY status a Monitor pass runs on.
- `fixing` — a fix agent is **in flight** pushing review/CI/conflict fixes; returns to
  `open`. Skip while in flight.
- `merged` — the task PR was merged **into its base** (integration, or its blocker's
  branch when stacked) — by the human, or by you only under an explicit user merge
  authorization; cleanup next.
- `done` — merged AND worktree/branch destroyed. Terminal (the task agent has died).
- `failed` — gate couldn't go green; PR left as draft with a blocker note for the human.

`specifying`/`implementing`/`fixing` imply `agentInFlight: true` (agent owns the task);
`pending`/`specified`/`open`/`merged` with `agentInFlight: false` are idle and get an
agent this tick.

`planPrStatus` (in `run.json`): `draft` (tasks in flight) → `ready` (all tasks merged,
un-drafted, awaiting human) → `merged` (human merged plan PR into `main` — run done).

## Field notes
- `runDir` / `runHash` — the located run directory and its 6-char token.
- `preflight` — BLOCKING gate (`ghAuth`, `mcp` per-server or `"none"`, `passedAt`);
  execution can't start until `passedAt` is set; re-verified before each new wave.
- `integrationBranch` — stacked base; every task branches from it and PRs target it.
- `integrationWorktree` — the checkout for `integrationBranch`. Its creation is where
  the cycle's plan/design docs are swept off `main` and committed (step 1.3), and where
  plan-PR CI/comment fixes run. Removed at cleanup once the plan PR merges.
- `planPr*` — the plan PR (integration → `main`); the human merges it last.
- `prTitlePattern` — `{exemplar, note}` PR-title convention; the orchestrator refreshes
  it from a `renamedTitle` reported by a task agent so new PRs match the latest style.
- `nextWakeupAt` — when `ScheduleWakeup` re-enters. Monitoring is in-session only — no
  cron / external scheduled agent.
- `baseBranch` — a task's PR base, resolved from its blockers: the `integrationBranch`
  (0 or 2+ blockers) or the **single blocker's branch** (stacked). Never `main`. Rebases
  and PR creation use this exact value; update it if a stacked base merges away (then
  re-base onto integration).
- `isDraft` — whether the task PR is a GitHub draft. A PR is draft while it isn't safe
  to merge — stacked on an unmerged base, CI not green, or an agent in flight — so a
  human can't merge it mid-flight. Un-drafted only when genuinely mergeable.
- `answeredComments` — `{commentId, threadId, verdict, replyUrl, threadResolved}` per
  handled comment. `replyUrl` is proof the reply posted (NO SILENT FIXES). For an
  agreed-and-fixed item, `threadResolved: true` records that the review thread was
  resolved (GraphQL `resolveReviewThread`) so an automated reviewer (e.g. Macroscope)
  sees it as solved — a reply alone doesn't signal resolution. Declined/clarify items
  have `threadResolved: false` (left open for the human). Recorded only after the reply
  is confirmed, so the agent never double-replies, misses one, or marks a fix handled
  without a visible reply + resolution.

## Resume logic (each wakeup — top orchestrator)
0. **Locate** the run dir (glob `*-<slug>-cycle/run.json`, match `planPath`) — never
   create a duplicate. Read `run.json` + every `tasks/<id>.json`.
1. Re-verify the preflight gate (auth/MCP can drop between sessions).
2. **Spawn one per-task agent per IDLE active task** (its **base branch exists** —
   integration, or its single blocker's branch when stacked; NOT "blockers merged" —
   `agentInFlight: false`) in a single message, model by phase: `pending`→**spec** (opus/high),
   `specified`→**implement** (`modelPolicy[complexity]`), `open`→**monitor** (sonnet),
   `merged`→**cleanup** (sonnet); **skip any task with `agentInFlight: true` or status
   `specifying`/`implementing`/`fixing`** (its agent owns it — don't touch its PR). Set
   `agentInFlight` when spawning; spec/implement/fix run in the background. Each agent
   resumes from its `tasks/<id>.json`, advances one step, writes its own file, returns
   a summary. The orchestrator does NO `gh`/fixing.
3. Clear `agentInFlight` for completed agents; update `run.json` from the summaries
   (cached `lastStatus`, `prTitlePattern` on a reported rename, plan-PR task lines
   added once on first open). Recover any agent past its stale lease.
4. **Dispatch (no merge gate):** any task whose **base branch now exists** becomes
   active next tick — a stacked child the moment its blocker's branch is pushed, not
   when it merges. Flow; never freeze a task waiting on another's merge.
5. When all tasks are `done`/`failed` → un-draft the plan PR (`planPrStatus = ready`);
   delegate plan-PR CI/comment fixes to an agent in the integration worktree.
6. If any task is active OR `planPrStatus ≠ merged` → **ScheduleWakeup again**
   (mandatory — turn-end invariant). Only once the plan PR is `merged` into `main`:
   remove the integration worktree/branch, write the summary, no wakeup.
