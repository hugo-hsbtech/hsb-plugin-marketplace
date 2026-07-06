---
name: cadence-planner
description: >
  Turn a list of tasks (or one task with sub-tasks) into a dependency-aware,
  parallel-safe development cycle plan that a project-management / orchestration
  agent can execute. Performs DEEP codebase analysis to detect real file/module
  dependencies and write-write conflicts, identifies which tasks are unblocked,
  and schedules tasks into parallel "waves". Plan-only — it hands off the schedule,
  it does NOT dispatch or implement.
  Use when the user says: "plan the cycle", "plan development cycles", "what can
  run in parallel", "which tasks are blocked / ready", "schedule these tasks",
  "wave plan", "parallelize these tickets", "build a cycle plan", "dependency
  analysis of these tasks", or pastes a task list / Linear issue / Jira issue /
  plan doc and asks how to run them in parallel. Also reachable via the
  /cadence:plan command, which passes inline task input straight into this skill.
  Do NOT use to actually implement tasks or dispatch agents — this only produces
  the plan.
---

# Planner — Parallel Development Cycle Planner

Convert a set of tasks into a **wave schedule**: groups of tasks that are safe to
implement in parallel, ordered so every task's blockers land before it starts.
The output is a handoff artifact for a separate orchestration/PM agent (or for
Murmur). **This skill plans; it does not implement or dispatch.**

## Core idea

Two tasks can run in the **same wave** only if BOTH hold:
1. **No ordering dependency** — neither needs an artifact (API, schema, type,
   module, migration) the other produces.
2. **No write-write conflict** — they don't edit the same file/region, or one
   doesn't structurally change a file the other reads in a way that breaks it.

A task is **unblocked** when all its dependencies sit in an earlier (already-planned)
wave. Wave 1 = every task with zero dependencies.

## Process — run these as todos

### 1. Ingest & normalize tasks — capture requirements LOSSLESSLY
Detect the input source and pull tasks into one normalized list. Sources:
- **Free-text / markdown** — parse the list; a task with indented bullets becomes
  a parent task + sub-tasks (sub-tasks are planned as individual schedulable units).
- **Linear** — use the Linear MCP / `ezra-check-linear-tickets` to fetch the issue
  and its sub-issues; capture Linear-declared blockers/relations.
- **Jira** — use the Jira MCP (search via ToolSearch if not loaded) to fetch the
  issue + sub-tasks + `blocks`/`is blocked by` links.
- **Plan doc** — read the file under `docs/plans/` (or the path given); each phase
  / numbered step becomes a task.

Normalize each task to:
```
ID:            T1, T2, …            (stable, referenced everywhere)
Title:         short imperative
Summary:       what "done" means (acceptance criteria if available)
Stated deps:   IDs the user/ticket explicitly declared as blockers
Source ref:    Linear/Jira key, file path, or "freetext"
Requirements:  [R…] atomic requirement checklist (see below) — the load-bearing field
```

**A one-line `Summary` is NOT enough capture and is where requirements get dropped.**
When the source is a rich plan/design doc, one plan *phase* becomes one task, and a
single Summary silently swallows everything buried inside that phase. Requirements
evaporate through three specific holes — capture against all three:

1. **Buried sub-sections.** A phase headline ("shim deletion, provenance, modal")
   hides numbered sub-parts (`§0/§1/§2`) and named paragraphs ("Pager total",
   "Inbox freshness", "Dismissed lifecycle") that are each their own requirement.
2. **Out-of-block requirements.** Requirements that live *outside* the phase bodies:
   the doc's **"Desired End State"** promises, any **design-doc / README promise the
   plan references** (e.g. "the confidence filter updates the stat counts"), the
   **"Decisions Made"** block, and the **"What we're NOT doing"** exclusion list.
3. **File-less requirements.** Requirements that don't map to a new file — cache
   invalidation, "endpoint X must honor filter Y", "read column Z too", an
   edge/error/empty/unknown-status state — have nothing to anchor them and vanish
   under a file-oriented analysis.

So decompose each task into an **atomic requirement checklist**. Every "Changes
Required" bullet, every numbered sub-section, every edge/error/empty/unknown-status
behavior, every acceptance criterion, plus the out-of-block requirements above,
becomes one atomic requirement with:
```
R-id:     T8.1, T8.2, …   (stable sub-id, task-scoped; referenced in the brief)
Text:     the concrete requirement, one behavior each
Anchor:   where it came from — "plan.md Phase 6 §2", "Desired End State",
          "design README", "What we're NOT doing"
```
Split compound requirements ("do A and also invalidate B") into separate R-ids —
the second clause is exactly the kind that drops. Also inventory the **exclusions**
(the source's NOT-doing / deferred items) as requirements tagged `excluded` with
their reason — an omission must become an owned decision, not a silent gap.

Keep a running **requirement ledger**: the flat list of every R-id inventoried
here. Step 5 must account for each one (assigned to a task, or on the NOT-doing
list) before the plan may be emitted.

If acceptance criteria are missing and a task is ambiguous, ask ONE batched round
of clarifying questions before analysis — don't guess scope on tasks that will
drive parallelization.

### 2. Deep codebase analysis (the expensive, important step)
For each task, determine its **touch set** — the concrete files/modules it will
create, edit, or depend on. Spawn analysis subagents in parallel (one per task,
or batch related tasks). **Run every analysis subagent on Opus, high effort** —
analysis is where the leverage is; don't cheap out on it (`Agent` with
`model: "opus"` and high reasoning effort; the `Explore` agent likewise). The
subagent is already deep-reading the source, so make it return the requirement
decomposition too — decomposition rides on work already happening, no extra hop.
Prompt:

> "For this task: «summary», sourced from «plan.md Phase N» (read the FULL phase
> body plus the doc's Desired End State, referenced design-doc/README promises, and
> the What-we're-NOT-doing list). Return two things:
> (1) A structured TOUCH SET: (a) files/modules likely CREATED, (b) files likely
> EDITED (with the function/region), (c) files/modules it READS or depends on,
> (d) any shared/global surface (router registration, DI container, migrations,
> generated types, index/barrel exports, config).
> (2) The ATOMIC REQUIREMENT CHECKLIST for this task: one line per concrete
> behavior — every Changes-Required bullet, every numbered sub-section, every named
> paragraph, every edge/error/empty/unknown-status state, and every out-of-block
> promise (Desired End State, referenced design-doc, exclusions) that this task
> owns. Split compound 'do A and also B' items in two. For each, give the text and
> a source anchor (phase §, or 'Desired End State' / 'design README' / 'NOT doing').
> Flag file-less requirements (invalidation, honor-a-filter, read-also-column,
> edge-state) explicitly — they are the ones that get dropped. Do not write code."

Collect per task:
```
creates:   [paths]
edits:     [path → region/function]
reads:     [paths/modules]
shared:    [migrations, barrel exports, router/DI wiring, codegen, lockfiles]
requirements: [R-id → text (anchor)]   ← reconcile with the Phase-1 ledger; if the
                                          subagent found a requirement Phase 1 missed,
                                          add it to the ledger with a new R-id
```
> The requirement list is not optional. A touch set with no `requirements` means the
> analysis only looked at files and will drop file-less requirements — send it back.
> Note: task **complexity / model tier is NOT decided here.** It is determined later
> by the executor during its superpowers spec/plan phase (real codebase analysis in
> context), not at plan/definition time. The planner only produces the touch set and
> schedule.
> `shared` surfaces (e.g. `__init__.py`/`index.ts` exports, Alembic migration
> chains, OpenAPI/Orval generated clients, `pnpm-lock.yaml`/`uv.lock`) are the most
> common hidden serializers — two tasks both appending to the same barrel or both
> creating migrations almost always conflict.

### 3. Build the dependency graph
Merge three dependency sources into a single directed graph (edge A→B = "B depends
on A / A must land first"):
- **Declared** — stated deps + Linear/Jira blocker links.
- **Producer→consumer** — task A `creates`/`edits` something task B `reads`
  (new API, type, schema, module, migration). A→B.
- **Write-write conflict** — A and B both `edit` the same file/region, or both
  touch the same `shared` surface. These have no inherent order, so impose an
  arbitrary deterministic order (lower ID first) to **serialize** them, and label
  the edge `conflict` (not a true logical dep).

Detect cycles. If a cycle exists, flag it explicitly — it means the tasks are
mutually entangled and should be merged into one task or manually split. Do not
silently break it.

### 4. Schedule into waves (topological levelling)
- Wave 1 = all tasks with in-degree 0 (no unmet deps).
- Remove them; recompute; the next in-degree-0 set is Wave 2; repeat.
- Within a wave, double-check no two tasks share a write-write conflict that the
  graph missed; if found, push the lower-priority one to the next wave.
- Note the **critical path** (longest dependency chain) — it sets the minimum
  number of cycles regardless of parallelism.

### 5. Emit the cycle plan (handoff artifact)
Write the plan to `docs/plans/proposed/` (or print inline if no repo plans dir) using
this filename convention:

```
docs/plans/proposed/<YYYYMMDD-HHMM>-<slug-of-proposed>-<task-id>.md
```
- **`<YYYYMMDD-HHMM>`** — generation timestamp from the system clock
  (`date +%Y%m%d-%H%M`), so plans sort chronologically and never collide.
- **`<slug-of-proposed>`** — kebab-case slug of the proposed cycle/work title.
- **`<task-id>`** — the originating **Linear/Jira issue key** (e.g. `ABC-1234`,
  uppercased). Use `cycle` when the input is free-text or spans multiple unrelated
  source issues.

Example: `docs/plans/proposed/20260625-1430-add-reply-followups-ABC-1234.md`

Record the same `slug`, `task-id` (source key), and generated timestamp in the
plan's metadata header (see the template) so the executor reads them from content,
not by parsing the filename.

Use the template in `references/cycle-plan-template.md`. The plan MUST contain: the
task table, the dependency graph (Mermaid), the wave schedule with parallel
groupings, a per-task **context brief** (touch set + **Requirements covered**
checklist + acceptance criteria) so a fresh implementing agent can start cold, a
**Scope boundary — NOT doing / deferred** section, conflict/serialization notes, the
critical path, and explicit handoff instructions.

**Completeness invariant — the plan may not be emitted until the ledger balances.**
Before writing the file, reconcile the Phase-1 requirement ledger against the briefs:
every inventoried R-id must appear in **exactly one** task's "Requirements covered"
checklist **or** in the "Scope boundary — NOT doing / deferred" section. Zero
requirements unaccounted, zero assigned twice. Emit the tally in the plan header
(`Requirements: N inventoried · N assigned · N deferred · 0 unaccounted`). If the
count doesn't balance, a requirement is being dropped — find where it belongs and
place it before emitting; never round the number to make it balance. This is
enumerate-then-assign bookkeeping done *as you build the briefs*, not a separate
review pass. Then STOP — do not implement.

## Output shape (summary; full template in references/)
```
TASKS           table: ID | title | source | deps
GRAPH           mermaid DAG
WAVES
  Wave 1 ║ parallel-safe ║ T1, T3, T5      ← all unblocked
  Wave 2 ║ parallel-safe ║ T2 (needs T1), T4
  Wave 3 ║               ║ T6 (needs T2,T4)
CRITICAL PATH   T1 → T2 → T6   (3 cycles minimum)
PER-TASK BRIEF  touch set + REQUIREMENTS COVERED (R-ids w/ anchors) + acceptance criteria
NOT DOING       scope boundary — excluded/deferred requirements + reason each
LEDGER          N inventoried · N assigned · N deferred · 0 unaccounted
CONFLICTS       serialized pairs + why
HANDOFF         "Dispatch tasks as their base branches appear and FLOW — don't gate
                a wave on the previous wave merging. A task with one blocker stacks
                its PR on that blocker's branch; 0/2+ blockers → base = integration."
```

## Guardrails
- **Plan only.** Never start implementing tasks or dispatch worker agents. The
  deliverable is the schedule.
- **Be conservative on conflicts.** When unsure whether two tasks conflict,
  serialize them. A wasted serial cycle is cheaper than two agents fighting over a
  file and producing a broken merge.
- **Surface, don't hide, uncertainty.** If codebase analysis couldn't resolve a
  task's touch set, mark it `analysis: low-confidence` and treat it as conflicting
  with everything in its area (serialize) until clarified.
- **Stable IDs.** Reuse the same task IDs in every section so the handoff agent can
  cross-reference.
- **Don't invent dependencies** from vibes — every edge must trace to a declared
  link, a producer→consumer relationship, or a concrete shared file.
- **Lose no requirement.** The plan is a completeness contract, not a summary. Every
  atomic requirement in the source — including buried sub-sections, out-of-block
  promises (Desired End State / referenced design doc), and file-less behaviors
  (invalidation, honor-a-filter, read-also-column, edge states) — must land in one
  task's "Requirements covered" or the "NOT doing" list. The header ledger must
  balance to `0 unaccounted`. Under-capturing a requirement is a worse failure than
  a wasted serial cycle: it ships as a silent bug or a missing feature.
