# PR body templates (cadence-executor)

Write for a human who has **not** been following the cycle. Be didactic and succinct:
say what changed and how to test it in plain language, and don't bury the reader in
internal cross-references. **Match the body's size to the task's `complexity`** — pick
the Simple body for a `low`/small change, the Full body only for genuinely complex work.

> **References must be legible.** Never point at a bare internal id (`T2b`, "wave 2",
> "the matcher task"). When you reference sibling work, give the **PR number + a
> one-line description** — "builds on #123 (the reply-correlation matcher)". If a task
> id must appear, gloss it once: "T2 (PR #123 — the inbound pipeline)".

---

## Open decisions block — goes at the TOP of either body, when any exist

The only section you keep updating after the PR is opened. Omit it entirely when there
are no open decisions. Each line links to the comment where the question is answered.

```markdown
## ⚠️ Open decisions

> These need a human. Answer in the linked comment (e.g. reply `D1: B`); this PR won't
> be marked ready or merged while a blocking one is open.

- **D1** (blocking) — Should expired invites be purged or archived? → [answer here](<commentUrl>) · *in effect: archive*
- ~~**D2** — Retry backoff: 3× or 5×?~~ → answered by @user: 5× · implemented in `abc1234`
```

---

## Simple body — use for `complexity: low` (and small `medium`)

For a one-file / low-risk change. A few sentences is the whole PR. **No Mermaid, no
acceptance-criteria checklist, no multi-section scaffolding.**

```markdown
<!-- If there are open decisions, the ⚠️ Open decisions block goes here, first. -->
## What & why

<1–3 plain sentences: what this changes and the problem it solves.>
<If it sits on another PR, say so on its own line: "Stacked on #123 (adds X) — merge
after it." / "Sits on a join of #123 + #124.">

## How to test

<2–4 plain steps, or a single line if it's that simple. Setup → action → expected.>

<!-- Decision log ONLY if a real autonomous choice was made — otherwise delete this. -->
## Decisions

- **<the choice>** — chose <X> over <Y>; to change it, comment "<how>".
```

---

## Full body — use for `complexity: high` (and rich `medium`)

For changes that span multiple files/services or carry real design decisions.

```markdown
<!-- If there are open decisions, the ⚠️ Open decisions block goes here, first. -->
## What & why

<1–3 sentences: the demand in plain language — what problem this solves and for whom.>

**Task:** <T-id> · **Source:** <Linear/Jira key | plan path> · **Cycle:** <slug>, wave <n>
**Base:** `<baseBranch>` — <plain note: "the integration branch", "stacked on #123
(adds X) — merge after it", or "a join of #123 + #124, whose conflicts are resolved here">
**Acceptance criteria**
- [ ] <criterion 1>
- [ ] <criterion 2>

## How it works

```mermaid
flowchart LR
  A[Trigger / entrypoint] --> B[Change]
  B --> C[Result]
```
<Add a sequence/architecture diagram only if the change spans services or async steps.>

<Short plain-language walkthrough of the approach and the key files touched.>

## UAT — how to test this

> Step-by-step so a non-author can verify it. Include setup + expected results.

1. **Setup:** <branch checkout / compose up / migrations / seed>
2. **Do:** <exact action — page to open, endpoint to call, command to run>
3. **Expect:** <observable result that proves it works>
4. **Edge case:** <one negative/edge path to try and its expected handling>

## Decision log (autonomous choices — roll back any of these)

> Succinct: one line per real choice — what was chosen, the alternative, and the
> off-ramp. Plain language, no implementation narration. Omit the table if there were
> no real choices.

| # | Decision | Chose (recommended) | Instead of | Roll back by |
|---|----------|---------------------|------------|--------------|
| 1 | <what>   | <chosen>            | <alt>      | comment "<how>" |

## Verification

- Lint/format: <result> · Tests: <suites + result> · Manual: <what you exercised, if any>

## Notes / risks

<Migrations, feature flags, follow-ups, anything the reviewer should watch. Keep short.>
```
