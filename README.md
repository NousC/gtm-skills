# gtm-skills

Free Claude Code skills for go-to-market teams — the deliverables behind the
[Nous](https://opennous.cloud) use cases.

One repo, focused on GTM. Each skill is a folder under `.claude/skills/`.

## Skills

| Skill | Use it for |
|-------|------------|
| [`linkedin-engagers`](.claude/skills/linkedin-engagers/SKILL.md) | Turn a creator's recent engagers into a deduped Nous lead list |
| [`cold-email-writer`](.claude/skills/cold-email-writer/SKILL.md) | Write a cold-email campaign that learns from past replies — pre-write suppression + pattern lookup baked in |

## Routines

Setup recipes for Claude Code routines — paste the prompt at
`claude.ai/code/routines`, attach the Nous MCP connector, schedule it.

| Routine | Use it for |
|---------|------------|
| [`chase-non-responders`](routines/chase-non-responders.md) | Every morning at 9am, draft a follow-up to leads who went quiet on email or LinkedIn — grounded in their full Nous account record |
| [`cold-reply-agent`](routines/cold-reply-agent.md) | When a cold-email reply lands in Instantly or Lemlist, classify it, pick the matching template, and send the right next message — via webhook |

## Install one skill

One command adds a skill to your Claude Code:

```bash
curl -sL https://raw.githubusercontent.com/NousC/gtm-skills/main/.claude/skills/linkedin-engagers/SKILL.md \
  --create-dirs -o ~/.claude/skills/linkedin-engagers/SKILL.md
```

## Use them all in a project

Clone the repo into the project where you want every skill available:

```bash
git clone https://github.com/NousC/gtm-skills
```

The `.claude/skills/` layout means Claude Code discovers every skill
automatically inside that project.

## License

MIT. Use them, fork them, ship them.
