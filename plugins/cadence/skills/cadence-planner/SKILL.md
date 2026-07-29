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
create, edit, or depend on.

**Ground the analysis in a code graph when available (graphifyy — optional
accelerator).** Before fanning out, check for the graphify CLI (`command -v
graphify`) or an existing `graphify-out/graph.json`. If present, build/refresh the
graph ONCE for the whole run — `graphify extract . --update` (local tree-sitter
parse: deterministic, no LLM cost, nothing leaves the machine) — and tell every
analysis subagent to query it FIRST: `graphify query "…"`, `graphify path A B`,
`graphify explain <Symbol>` (or the graphify MCP tools). The graph answers
imports/calls/inheritance/dependents deterministically, so subagents read only the
files the graph points at instead of exploring by trial and error — it is also the
strongest evidence for producer→consumer edges and shared-surface conflicts in
step 3. If graphify is absent, run the analysis exactly as below by reading files —
**graphify is never a requirement, and its absence never blocks or degrades the
plan's completeness bar.**

Spawn analysis subagents in parallel (one per task,
or batch related tasks). **Run every analysis subagent on Opus, high effort** —
analysis is where the leverage is; don't cheap out on it (`Agent` with
`model: "opus"` and high reasoning effort; the `Explore` agent likewise). The
subagent is already deep-reading the source, so make it return the requirement
decomposition too — decomposition rides on work already happening, no extra hop.
Prompt (when graphify is available, prepend: "A code knowledge graph exists —
query it with `graphify query/path/explain` before reading files, and cite graph
edges as evidence in the touch set"):

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
  (new API, type, schema, module, migration). A→B. When a graphify graph exists,
  its import/call/dependent edges are the preferred evidence for these links —
  cite the edge instead of inferring from prose.
- **Write-write conflict** — A and B both `edit` the same file/region, or both
  touch the same `shared` surface. These have no inherent order, so impose an
  arbitrary deterministic order (lower ID first) to **serialize** them, and label
  the edge `conflict` (not a true logical dep).

Detect cycles. If a cycle exists, flag it explicitly — it means the tasks are
mutually entangled and should be merged into one task or manually split. Do not
silently break it.

### 3b. Bundle pass — fold trivial prep work into ONE task per class
Per-task overhead is fixed and mostly independent of the change: a worktree, an
Opus-tier spec agent, a PR, a CI run, a review round-trip, and one of the human's merge
clicks. Spending all of that on a doc typo is waste, and eight trivial PRs spend the
human's scarcest resource — attention — eight times. A **prep bundle** is one task
carrying several small, same-class scopes, scheduled early so it merges first and gives
every other task a clean base.

**Bundling is a PLAN-TIME decision.** The executor never folds one task into another's
PR — it can't (a task agent may not touch a sibling's branch). A bundle only exists if
you emit it as a task here, with a **union touch set**.

**Classes — never mix them in one bundle:**

| Class | Contains | Review bar |
|---|---|---|
| `docs` | documentation, comments, README/changelog, lint/formatting — **no behavior change whatsoever** | light; the gate is enough |
| `config` | config files, scaffolding, dependency pins, CI/workflow tweaks — no product logic | normal |
| `schema` | **migrations only** — bundled *with each other*, giving one revision chain instead of competing `down_revision`s | full; never light |

Mixing classes silently raises the whole PR to the riskiest item's level: a doc typo
drags through a careful review, or a migration slides through a cursory one. **A
migration never joins a `docs` bundle.**

**An item is eligible only if ALL hold:**
1. **Small and low-risk on inspection** — a candidacy judgement, not a complexity
   assignment. (`complexity` remains the executor spec phase's call, for the bundle as a
   whole; a bundle inherits its **highest** scope's complexity.)
2. **Same class** as the rest of the bundle.
3. **No write-write conflict** among the bundled items, and the union creates no new
   conflict with another task.
4. **Its dependents can live with bundle-granularity.** If another task depends on item
   A, that edge is re-pointed to the whole bundle. Only bundle A if the bundle is small
   and lands in an early wave — never make half the cycle wait behind a bundle carrying
   something debatable.
5. **Caps:** ≤ 5 scopes and ≤ ~15 files per bundle. Beyond that, emit a second bundle of
   the same class rather than one unreviewable PR.

**Emitting one:**
- Give it a normal task id, titled `prep(<class>): <plain summary>` (e.g.
  `prep(docs): 4 doc + comment fixes`), and schedule it in the **earliest wave it
  qualifies for** so it merges first.
- List every scope as `T<id>.<n>` with its own one-line what/why and **its own R-ids** —
  the requirement ledger must still balance (`0 unaccounted`); a bundle must never
  swallow a requirement's traceability.
- Record `Bundle: <class>` plus the per-scope breakdown in the task's context brief, and
  note the union touch set.
- **Say it's reversible as a set.** One revert takes all the scopes with it — call that
  out so the human can ask for a split before shipping.
- **When in doubt, don't bundle.** A separate PR is always a legitimate outcome; a
  confusing multi-scope PR is not.

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

**READ `references/cycle-plan-template.md` IN FULL BEFORE WRITING A SINGLE LINE OF THE
PLAN.** Not "consult" — read it, then follow its section order and its per-section
shapes exactly. The summary at the bottom of this file is an index, not a substitute:
working from the summary alone is how a plan ends up with hand-drawn ASCII where the
template mandates Mermaid, and prose blobs where it mandates field sets.

> ### THE BRIEF IS A FIELD SET, NOT PROSE (the most expensive failure this file has)
> A per-task brief is **structured fields** — Goal/done · **Requirements covered**
> (R-id checklist with source anchors) · Creates · Edits · Reads/depends · Shared
> surfaces · Blocks · Notes. Writing the same knowledge as one dense paragraph is a
> **defect**, even when the paragraph is brilliant — and it usually is, because the
> analysis was real. What the paragraph destroys is everything downstream that reads
> the *fields*:
> - **the requirement ledger** — no R-ids means no proof that nothing was dropped
>   between the source and the plan, which is the entire point of Phase 1;
> - **the executor's scope check** — it compares a PR's diff against the task's
>   Creates/Edits touch set; prose gives it nothing to check, so diff pollution goes
>   undetected;
> - **verify-and-extend** — a spec agent can verify a field list against current code;
>   it cannot verify a paragraph, so it re-derives the whole analysis and the cycle
>   pays for it twice.
>
> Hard-won detail ("three test surfaces pin this positionally…", "hunt every `13`")
> belongs in **Notes** — *in addition to* the fields, never *instead of* them.

**Never compress the plan to save space.** Plan length scales with task count: 16 tasks
means a longer document than 8, not a denser one. If it feels too big, that is a signal
to **split the cycle**, not to thin the briefs — a short plan for a large cycle is the
symptom this rule exists to catch.

Use the template in `references/cycle-plan-template.md`. The plan MUST contain: the
task table, the dependency graph (Mermaid), the wave schedule with parallel
groupings, a per-task **context brief** (touch set + **Requirements covered**
checklist + acceptance criteria) so a fresh implementing agent can start cold —
the executor's spec phase **consumes this brief directly** (it verifies and
extends it rather than re-deriving the analysis), so the brief's completeness is
what spares the executor from paying for this analysis a second time — plus a
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
review pass.

> ### PRE-WRITE GATE — run these assertions before the file is written
> Mechanical, not advisory. Check each one against the text you are about to write; any
> failure is fixed **before** writing, not noted afterwards.
> 1. **Every task brief has an R-id checklist** with at least one `**T<id>.<n>**` entry,
>    each carrying a source anchor. Zero R-ids anywhere in the document = stop.
> 2. **The ledger balances and is in the header:**
>    `assigned + deferred = inventoried`, `0 unaccounted`.
> 3. **Every brief carries its fields** — Goal/done, Requirements covered, Creates,
>    Edits, Blocks at minimum. A brief that is one paragraph of prose fails this.
> 4. **The dependency graph section contains a ```mermaid block.** No hand-drawn
>    diagrams: the document must contain **no box-drawing characters** (`─ │ ┌ ┐ └ ┘ ├
>    ┤ ┬ ┴ ┼ ► ▼ ◄ ▲`) anywhere. If a graph feels too complex for Mermaid, that is what
>    dashed conflict edges, edge labels, and wave subgraphs are for — see the template.
> 5. **Section order matches the template**, and no mandated section is missing
>    (tasks · graph · waves · briefs · conflicts · scope boundary · handoff).
>
> If you cannot satisfy one of these, say so explicitly in the plan and in your summary
> — never emit a plan that quietly fails a gate.

Then STOP — do not implement.

## Output shape (summary; full template in references/)
```
TASKS           table: ID | title | source | deps | bundle | confidence
GRAPH           MERMAID ONLY (```mermaid graph LR) — dashed labelled edges for
                write-write conflicts, subgraph per wave. ASCII art is a DEFECT.
WAVES
  Wave 1 ║ parallel-safe ║ T1, T3, T5      ← all unblocked
  Wave 2 ║ parallel-safe ║ T2 (needs T1), T4
  Wave 3 ║               ║ T6 (needs T2,T4)
CRITICAL PATH   T1 → T2 → T6   (3 cycles minimum)
PREP BUNDLES    prep(<class>) tasks folding trivial same-class work into one PR (or "none")
PER-TASK BRIEF  FIELDS, never a prose paragraph:
                  Goal/done · Requirements covered (R-id checklist + anchors) ·
                  Creates · Edits · Reads/depends · Shared surfaces · Blocks · Notes
                (hard-won detail goes in Notes, IN ADDITION to the fields)
NOT DOING       scope boundary — excluded/deferred requirements + reason each
LEDGER          N inventoried · N assigned · N deferred · 0 unaccounted
CONFLICTS       serialized pairs + why
HANDOFF         "Dispatch tasks as their base branches appear and FLOW — don't gate
                a wave on the previous wave merging. A task with one blocker stacks
                its PR on that blocker's branch; a task with 2+ blockers gets a join
                branch (integration + all blockers) so it starts too; no blockers →
                base = integration."
```

## Guardrails
- **Plan only.** Never start implementing tasks or dispatch worker agents. The
  deliverable is the schedule.
- **Bundle conservatively, and never across classes.** A prep bundle exists to spare
  the human eight merge clicks on trivial work — not to shrink the PR count. Same class
  only (`docs` / `config` / `schema`), ≤5 scopes, ≤~15 files, no write-write conflicts,
  every scope keeping its own R-ids. A migration never rides along with docs. When in
  doubt, emit separate tasks.
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
