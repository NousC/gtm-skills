# gtm-skills

Free Claude Code skills for go-to-market teams — the deliverables behind the
[Nous](https://opennous.cloud) use cases.

One repo, focused on GTM. Each skill is a folder under `.claude/skills/`.

## Skills

| Skill | Use it for |
|-------|------------|
| [`linkedin-engagers`](.claude/skills/linkedin-engagers/SKILL.md) | Turn a creator's recent engagers into an ICP-scored Nous lead list, saved to Nous and a Google Sheet |
| [`meeting-brief`](.claude/skills/meeting-brief/SKILL.md) | Before a meeting, pull what Nous knows, read their recent posts and company site, and write a sourced brief grounded in your GTM angle |
| [`morning-brief`](.claude/skills/morning-brief/SKILL.md) | Each morning, cross your Timestripe goals with your Nous funnel into one accountable brief — what to work first, where the funnel stands, and a 1% Kaizen logged to Notion |
| [`client-report`](.claude/skills/client-report/SKILL.md) | Write a weekly, client-ready report from a client's Nous workspace — lead lists, who engaged, the funnel, what's converting — into a document, ending in next week's moves |
| [`hiring-signals`](.claude/skills/hiring-signals/SKILL.md) | Discover companies hiring the role you sell into (TheirStack), score them against your Nous ICP, dedupe against your pipeline, record the signal, and draft a cold sequence that opens on it |
| [`lead-builder`](.claude/skills/lead-builder/SKILL.md) | Describe your ICP, give example companies, or name one person → Claude builds a precise targeting spec, runs it through Apollo's free people search, reveals + verifies emails (FullEnrich + MillionVerifier), dedupes against Nous, saves to a lead list. No Sales Navigator needed. Three phases: shape the spec, confirm ICP + cost, run |
| [`sales-nav-builder`](.claude/skills/sales-nav-builder/SKILL.md) | Turn a LinkedIn Sales Navigator search into a clean, ICP-scored lead list, paying for emails only on the leads you keep. Claude builds the search from structural filters first, Evaboot extracts leads with no emails, Nous excludes the wrong types and scores against your ICP, then emails are found only on qualified net-new leads. Non-ICP leads stay labelled. For Sales Nav users |
| [`clone-winners`](.claude/skills/clone-winners/SKILL.md) | The closed-won flywheel: pull the deals that actually closed → lookalike companies of them → enrich the buyers → read the sequence that won them → write a new campaign in that pattern, all saved to Nous |
| [`campaign-writer`](.claude/skills/campaign-writer/SKILL.md) | Write a full outbound sequence grounded in your GTM profile and what's actually replied — learns the winning variant, suppresses who you've touched, records the copy back |

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
