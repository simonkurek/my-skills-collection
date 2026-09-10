# my-skills-collection

A Claude Code plugin marketplace for **domain modeling**: mapping requirements onto
software archetypes, and distilling bounded contexts.

## Install

```bash
# in Claude Code
/plugin marketplace add simonkurek/my-skills-collection
/plugin install archetype-modeling@my-skills-collection
```

To develop against a local checkout instead:

```bash
/plugin marketplace add /path/to/my-skills-collection
```

## Plugins

| Plugin | Skills | What it does |
|--------|--------|--------------|
| [`archetype-modeling`](plugins/archetype-modeling) | 5 | Archetype mappers, a parallel archetype scanner, a mapper-skill builder, and a bounded-context distiller |

## Repository layout

```
.claude-plugin/marketplace.json     # marketplace manifest — lists every plugin
plugins/<plugin>/
  .claude-plugin/plugin.json        # plugin manifest
  skills/<skill>/SKILL.md           # one directory per skill
```

Adding a skill means adding a `skills/<name>/SKILL.md` under a plugin — the plugin
manifest does not enumerate skills. Adding a plugin means a new `plugins/<name>/`
directory plus a row in `plugins` in the marketplace manifest.
