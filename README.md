# hsb — Claude Code plugins

Hugo Seabra's Claude Code plugin marketplace. It ships **Cadence** — a
**plan → ship** pipeline for dependency-aware, parallel-safe development cycles.

## Install

In Claude Code:

```
/plugin marketplace add hugo-hsbtech/hsb-plugin-marketplace
/plugin install cadence@hsb
```

The first command registers this marketplace (named `hsb`); the second installs the
`cadence` plugin from it. Pull updates later with `/plugin marketplace update hsb`.

## Cadence — what's inside

| Command | Skill | What it does |
|---|---|---|
| `/cadence:plan <tasks>` | `cadence-planner` | Analyzes a task list (or a Linear/Jira issue) against your codebase, computes each task's touch set + real dependencies, and schedules the work into parallel **waves**. Plan-only — it writes a cycle-plan doc and stops. |
| `/cadence:ship <plan>` | `cadence-executor` | Consumes a cycle-plan doc and drives every task to a merged PR — one worktree, branch, and PR per task, stacked by dependency — monitoring reviews, CI, and conflicts on a scheduled loop until a human merges. |

### `/cadence:plan`

Pass a task list, a tracker key, or a plan-doc path:

```
/cadence:plan
- Add reply-correlation matcher
- Wire matcher into the inbound pipeline (depends on the matcher)
- Add a metrics dashboard
```

It computes dependencies and write-write conflicts, levels the tasks into waves, and
writes a wave schedule to `docs/plans/proposed/<timestamp>-<slug>.md`. It never
implements or dispatches — planning only.

### `/cadence:ship`

```
/cadence:ship docs/plans/proposed/20260706-1200-my-cycle.md
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
.claude-plugin/marketplace.json    the `hsb` marketplace → the cadence plugin
plugins/cadence/
  .claude-plugin/plugin.json        plugin manifest (name / version / author)
  commands/                         /cadence:plan, /cadence:ship (thin entrypoints)
  skills/
    cadence-planner/                  planning behavior + plan-doc template
    cadence-executor/                 execution behavior + state schema + PR template
```

The plugin is entirely Markdown — there is no build, lint, or test step.

## Author

Hugo Seabra
