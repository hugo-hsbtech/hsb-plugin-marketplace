---
description: Render the evidence report for a Cadence cycle — what shipped, what needed you, what went wrong, and what it cost — as one pasteable file. Works on a finished run or an in-flight one.
argument-hint: "<plan path, cycle slug, or run dir — leave empty to use the most recent run in .cadence/cycles/>"
---

Produce the **cycle report** for a Cadence run: a single self-contained markdown file
the user can read, paste into a chat, or hand to the plugin's maintainer as the evidence
record of how the cycle actually went.

## Which run

```
$ARGUMENTS
```

Resolve it, in this order:

- **A run dir** (`.cadence/cycles/<...>-cycle/`) → use it.
- **A plan path** → glob `.cadence/cycles/*-cycle/run.json` and pick the one whose
  `planPath` matches.
- **A slug** → glob `.cadence/cycles/*-<slug>-cycle/run.json`; newest datetime prefix wins.
- **Empty** → the most recently modified run dir under `.cadence/cycles/`. If there are
  several and it's ambiguous, list them with their slug, status, and date, and ask which.
- **No run dirs at all** → say so plainly (nothing to report on) and stop. Do not invent
  a report.

## What to do

1. Read **`${CLAUDE_PLUGIN_ROOT}/skills/cadence-executor/references/cycle-report.md`**
   first — it defines the event schema, the template, and the rules. Follow it exactly.
2. Read the run's `run.json`, every `tasks/<id>.json`, and every `events/*.jsonl`; merge
   the events by timestamp.
3. Fill in the gaps from GitHub **read-only** where the logs are thin — `gh pr view` /
   `gh api graphql` for PR states, merge facts, and review timings. Never post, edit,
   merge, or otherwise write to a PR from this command: it is a reporting command.
4. Render the report to **`<runDir>/report.md`** and print: the path, the Outcome block,
   **what needed you**, and **what went wrong**. Mark it `IN FLIGHT` if the run isn't
   finished.

## Non-negotiables

- **Evidence or nothing.** Every line anchors to a PR number, comment URL, commit SHA,
  or timestamp. A metric with no logged event is **`not captured`** — never estimated,
  never inferred, never rounded into existence.
- **Don't flatter the run.** "What went wrong" is the section the report exists for, and
  it must include **Cadence's own** failures — stalls, `waiting-for-merge` conversions,
  re-drafts after a PR was marked ready, parked review loops, model escalations,
  recovered stale leases, blocked auto-merge guards, decisions that shipped unresolved.
  An empty failure section on a messy run is a broken report, not a clean one.
- **Read-only.** This command writes exactly one file (`report.md`) and touches nothing
  else — not the state files, not the event logs, not a branch, not a PR.
