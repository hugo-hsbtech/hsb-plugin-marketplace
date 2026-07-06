---
description: Plan a dependency-aware, parallel-safe development cycle from a task list, Linear/Jira issue, or plan doc. Produces a wave schedule (plan-only handoff).
argument-hint: "<task list | Linear key | Jira key | path to plan doc>  (leave empty to be prompted)"
---

Load and follow the `cycle-planner` skill (plugin `hsb`) to produce a
dependency-aware, parallel-safe development **cycle plan**. This is a **plan-only
handoff** — do NOT implement tasks or dispatch worker agents.

## Input

The user's task input is below (may be empty):

```
$ARGUMENTS
```

Resolve the input source before planning:
- **Empty** → ask the user to paste a task list, or give a Linear key, Jira key,
  or a path to a plan doc. Do not proceed without tasks.
- **Looks like a Linear key** (e.g. `ABC-1234`) → fetch the issue + sub-issues and
  their blocker links via the Linear MCP / `ezra-check-linear-tickets`.
- **Looks like a Jira key** (e.g. `PROJ-123`) → fetch the issue + sub-tasks + link
  relations via the Jira MCP (load it with ToolSearch if not active).
- **Looks like a file path** (contains `/` or ends in `.md`) → read that plan doc.
- **Otherwise** → treat as a free-text / markdown task list and parse it.

## What to do

Run the full `cycle-planner` skill workflow on the resolved tasks:
1. Normalize tasks to stable IDs (T1, T2, …) with summaries and stated deps.
2. **Deep codebase analysis** — spawn parallel `Explore` subagents to compute each
   task's touch set (creates / edits / reads / shared surfaces).
3. Build the dependency graph (declared + producer→consumer + write-write conflict).
4. Schedule into parallel **waves** via topological levelling; compute the critical path.
5. Emit the cycle plan using the skill's `references/cycle-plan-template.md`, writing
   to `docs/plans/proposed/<YYYYMMDD-HHMM>-<slug-of-proposed>-<task-id>.md` when a
   repo plans dir exists (`<task-id>` = source Linear/Jira key, or `cycle` for
   free-text). Record slug + task-id + timestamp in the plan's metadata header.

Then STOP at the plan and present the wave schedule + handoff instructions. Be
conservative: when two tasks might conflict, serialize them.
