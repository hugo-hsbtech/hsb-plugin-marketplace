# 2026-07-29 — plan-doc fidelity

**Source:** live cycles in `pagana-platform/pagana-catalog-apps`.
Reference (good): `docs/plans/proposed/20260727-2142-voucher-phase1-domain-core-cycle.md` (on `main`).
Regression (poor): `docs/plans/proposed/20260728-2258-voucher-phase2-c2-statements-dashboards-cycle.md` (PR #56).
**Plugin version at run time:** not confirmed — see F3.

## Findings

### F1 — the dependency graph was ASCII art where the template mandates Mermaid
**Class:** behavior defect (compliance, not a template gap).

Evidence: PR #56's plan doc, §2 "Dependency graph" — a hand-drawn box-character diagram
with edge annotations crammed into the art, followed by a paragraph titled *"Notes on
edges the numbers hide"*. `cycle-plan-template.md` §2 already prescribed a ```mermaid
`graph LR` with a legend and a dashed-edge convention for write-write conflicts, and
`SKILL.md` step 5 already said "the dependency graph (Mermaid)".

So the instruction existed and was not followed. The "Notes on edges the diagram hides"
paragraph is the tell: the chosen format couldn't carry the nuance, and the planner
patched around it rather than switching format.

### F2 — per-task briefs collapsed from field sets to prose, and the requirement ledger vanished
**Class:** behavior defect (same root cause as F1).

| | phase 1 (rich) | cycle 2 (poor) |
|---|---|---|
| Plan length | 481 lines | 178 lines |
| Tasks | 14 | 16 (more work, ⅓ the doc) |
| Atomic requirements (R-ids) | 137 | **0** |
| Ledger | `149 inventoried · 137 assigned · 12 deferred · 0 unaccounted` | absent |
| Per-task brief | Goal/done · Requirements covered · Creates · Edits · Blocks | one free-text paragraph |

**Not laziness.** Cycle 2's briefs carry real, expensive analysis ("three test surfaces
pin the wiring positionally… `toHaveLength(3)`→4, `servers[0..2]`→`[0..3]`"). The
planner did the work and discarded the *structure*, which is what everything downstream
reads:

- **the completeness contract** — no R-ids means no proof nothing was dropped between
  the source doc and the plan, the entire purpose of the planner's Phase 1;
- **the executor's scope check** (v3.5.0) — it compares a PR's diff against the task's
  Creates/Edits touch set; prose gives it nothing to check against, so diff pollution
  goes undetected;
- **verify-and-extend** — a spec agent can verify a field list against current code but
  not a paragraph, so it re-derives the analysis and the cycle pays for it twice.

Root cause for both F1 and F2: the requirements lived in prose in the middle of a long
step and in a terse "Output shape" summary (`GRAPH  mermaid DAG`). Descriptive text is
easy to read past; nothing forced the template to be read or the output to be checked.

### F3 — plan PR #56's body carries no cycle diagrams
**Class:** unverified.

v3.9.0 (21:22) added cycle diagrams to the plan PR body; the plan doc is stamped 22:58 —
close enough that the run may have been on a pre-3.9.0 install. **Deferred**: needs the
run's `policyVersion` / report header. Not "fixed" while it may never have been running.

## Changed

All in `cadence-planner`, unreleased (no version bump — see the versioning policy):

1. **`SKILL.md` step 5 — read the template first.** A hard instruction ("READ … IN FULL
   BEFORE WRITING A SINGLE LINE"), stating that the Output-shape summary is an index,
   not a substitute, and naming both failure modes it produces.
2. **`SKILL.md` — "THE BRIEF IS A FIELD SET, NOT PROSE"**, with the three downstream
   consumers it breaks, and the rule that hard-won detail goes in **Notes** *in addition
   to* the fields.
3. **`SKILL.md` — "Never compress the plan to save space."** Length scales with task
   count; a short plan for a large cycle is the symptom. If it feels too big, split the
   cycle, don't thin the briefs.
4. **`SKILL.md` — PRE-WRITE GATE**, five mechanical assertions run against the text
   before the file is written: R-id checklist per brief · ledger balances · briefs carry
   their fields · a ```mermaid block and **no box-drawing characters anywhere** ·
   section order matches the template.
5. **`SKILL.md` output shape** — `GRAPH  mermaid DAG` replaced with "MERMAID ONLY …
   ASCII art is a DEFECT"; `PER-TASK BRIEF` expanded to the explicit field list.
6. **`cycle-plan-template.md`** — the Mermaid example now demonstrates what tempted the
   ASCII: wave subgraphs, labelled dashed conflict edges, multi-parent fan-in, and
   `id · what it does` node labels. The Notes field explicitly welcomes dense prose *in
   addition to* the fields. Verified by rendering with mermaid-cli.

## Not changed (deliberately)

- **An executor-side refusal to ship a plan with zero R-ids.** Tempting — it would have
  caught this at `/cadence:ship` instead of at review. Deferred so this round changes
  one thing and we learn whether the planner-side gate is enough; a hard executor
  refusal would also block a legitimately hand-written plan. **Evidence that would
  settle it:** if the next cycle's plan still emits without R-ids, add the gate at ship
  time too.
- **A verification pass over the emitted plan by a second agent.** Real cost per cycle;
  the pre-write gate is deterministic and free. Revisit only if the gate is observed
  being skipped.
- **PR #56's existing plan doc.** Already shipped in the target repo; not ours to
  rewrite. Future plans get the fix.
- **F3.** Unverified — see above.
