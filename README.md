# hsb — Claude Code plugin marketplace

Hugo Seabra's personal Claude Code plugins. Ships one plugin, **`hsb`**, a
**plan → execute** pipeline for dependency-aware, parallel-safe development cycles.

## Install

In Claude Code:

```
/plugin marketplace add hugo-hsbtech/hsb-plugin-marketplace
/plugin install hsb@hsb
```

The first command registers this marketplace; the second installs the `hsb` plugin
from it. Pull updates later with `/plugin marketplace update hsb`.

## What's inside

| Command | Skill | What it does |
|---|---|---|
| `/hsb:planner <tasks>` | `cycle-planner` | Analyzes a task list (or a Linear/Jira issue) against your codebase, computes each task's touch set + real dependencies, and schedules the work into parallel **waves**. Plan-only — it writes a cycle-plan doc and stops. |
| `/hsb:execute-cycles <plan>` | `cycle-executor` | Consumes a cycle-plan doc and drives every task to a merged PR — one worktree, branch, and PR per task, stacked by dependency — monitoring reviews, CI, and conflicts on a scheduled loop until a human merges. |

### `/hsb:planner`

Pass a task list, a tracker key, or a plan-doc path:

```
/hsb:planner
- Add reply-correlation matcher
- Wire matcher into the inbound pipeline (depends on the matcher)
- Add a metrics dashboard
```

It computes dependencies and write-write conflicts, levels the tasks into waves, and
writes a wave schedule to `docs/plans/proposed/<timestamp>-<slug>.md`. It never
implements or dispatches — planning only.

### `/hsb:execute-cycles`

```
/hsb:execute-cycles docs/plans/proposed/20260706-1200-my-cycle.md
```

It opens an **integration branch** + a draft plan PR, then one PR per task (stacked on
its blocker, or targeting integration), and keeps monitoring until you merge.

> **Safety rails:** it **never pushes or commits to `main`** and **never merges a PR** —
> merging both task PRs and the plan PR is always your gate.

## Requirements

- **Claude Code** with plugin support.
- **`gh` CLI**, authenticated — the executor opens PRs and posts review replies with it.
- **The `superpowers` skill library** (installed as a Claude Code plugin) — the skills
  build on `brainstorming`, `writing-plans`, `test-driven-development`,
  `using-git-worktrees`, `verification-before-completion`, and others.
- When tasks are linked to a tracker (Linear/Jira/…), that tracker's MCP server must be
  connected with write access so status can be mirrored as the cycle runs.

## Repository layout

```
.claude-plugin/marketplace.json    this marketplace → the hsb plugin
plugins/hsb/
  .claude-plugin/plugin.json        plugin manifest (name / version / author)
  commands/                         /hsb:planner, /hsb:execute-cycles (thin entrypoints)
  skills/
    cycle-planner/                  planning behavior + plan-doc template
    cycle-executor/                 execution behavior + state schema + PR template
```

The plugin is entirely Markdown — there is no build, lint, or test step.

## Author

Hugo Seabra · <contato.hsbtec@gmail.com>
