# ucgmsim skills

Claude Code skills for working on ucgmsim repositories, packaged as a plugin marketplace.

## Install

In Claude Code:

```
/plugin marketplace add ucgmsim/skills
/plugin install ucgmsim@ucgmsim
```

Skills are then available as `/ucgmsim:<skill>`.

## Skills

| Skill | Purpose |
| --- | --- |
| `release` | Cut a GitHub release (and trigger the PyPI publish) for a ucgmsim repo, with release notes in the house style. |

## Adding a skill

Add `plugins/ucgmsim/skills/<name>/SKILL.md` with `name` and `description` frontmatter, list it in the table above, and bump `version` in `plugins/ucgmsim/.claude-plugin/plugin.json`.
