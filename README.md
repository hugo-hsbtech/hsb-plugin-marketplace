# hsb — Claude Code plugins

Hugo Seabra's Claude Code plugin marketplace.

## Add the marketplace

```
/plugin marketplace add hugo-hsbtech/hsb-plugin-marketplace
```

Then install any plugin below with `/plugin install <name>@hsb`.

## Plugins

| Plugin | Install | What it does | Docs |
|---|---|---|---|
| **Cadence** | `/plugin install cadence@hsb` | A plan → ship pipeline for dependency-aware, parallel development cycles. `/cadence:plan` schedules tasks into parallel waves; `/cadence:ship` drives each to a merged PR; `/cadence:report` renders the run's evidence report. **Requires the `superpowers` plugin**; optional [`graphifyy`](https://github.com/Graphify-Labs/graphify) for cheaper analysis. | [plugins/cadence](plugins/cadence/README.md) |

Each plugin ships its own deep-dive documentation — follow the **Docs** link for
concepts, command reference, and examples.

## Repository layout

```
.claude-plugin/marketplace.json    the `hsb` marketplace manifest → lists the plugins
plugins/<name>/                    one directory per plugin
  README.md                        the plugin's deep-dive documentation
  .claude-plugin/plugin.json       plugin manifest (name / version / author)
  commands/                        slash-command entrypoints
  skills/                          the behavior (SKILL.md + references)
```

```
CHANGELOG.md                       releases per plugin; work in progress sits under
                                   "Unreleased" until a batch is shipped
docs/feedback/<date>-<slug>.md     one file per improvement round: the findings from a
                                   real run, what changed, and what deliberately didn't
```

The plugins are entirely Markdown — there is no build, lint, or test step.

## Improving a plugin from a real run

These plugins evolve from evidence, not hunches. Finish a cycle, render its report
(`/cadence:report`), then bring it here — paste it, or hand over the file. The session is
**interactive**: findings are traced from your evidence to the exact rule that caused
them, you're asked which ones to take and how (and asked explicitly before anything
touches a safety invariant), the plan is shown before edits land, and the round is
recorded under `docs/feedback/` with what changed *and what didn't, with reasons*. See
[the loop in detail](plugins/cadence/README.md#closing-the-loop-the-report-is-how-cadence-evolves).

## Author

Hugo Seabra
