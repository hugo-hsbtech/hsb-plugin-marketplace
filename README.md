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
| **Cadence** | `/plugin install cadence@hsb` | A plan → ship pipeline for dependency-aware, parallel development cycles. `/cadence:plan` schedules tasks into parallel waves; `/cadence:ship` drives each to a merged PR. | [plugins/cadence](plugins/cadence/README.md) |

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

The plugins are entirely Markdown — there is no build, lint, or test step.

## Author

Hugo Seabra
