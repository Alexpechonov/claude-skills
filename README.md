# claude-skills

Personal [Claude Code](https://claude.com/claude-code) skills.

Cloned to `~/.claude/skills`, where Claude Code picks them up automatically.

| Skill | What it does |
| --- | --- |
| [adversarial-review](adversarial-review/SKILL.md) | Spawns a read-only agent to break work you just produced, then triages its findings and applies the fixes yourself. |

## Adding a skill

One directory per skill, each with a `SKILL.md` carrying YAML frontmatter:

```yaml
---
name: my-skill
description: What it does, and the phrasings that should trigger it.
---
```

Everything after the frontmatter is the instruction body Claude follows when the skill
is invoked, either by name (`/my-skill`) or because the description matched the request.
