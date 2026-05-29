---
name: cold-email-writer
description: Design and write cold-email campaigns that learn from your own data. Reads the Nous workspace first — who's already touched, who replied, who said stop, and what tone/length/CTA has worked for this segment before — then writes the sequence in the pattern that's winning. Use when designing outbound for a new segment, refreshing a campaign that has gone flat, or running outbound across a list where you've burned cycles before.
---

# Cold Email Writer

## What it does

Most cold email gets written from a template — usually a hand-me-down
playbook from someone else's stack. The template doesn't know which tone
landed for *your* segment last quarter, which length got opens, which CTA
got replies, which lead source needs which angle. So every campaign starts
from zero, and reply rates stay flat.

This skill closes the loop. Before writing a single line:

1. It calls Nous to **suppress** anyone already contacted / replied /
   bounced / unsubscribed / DNC'd across every list you've ever run.
2. It calls Nous to **learn** — what tone, length, angle, and CTA have
   actually produced replies for this segment in your past campaigns.
3. It calls Nous to **contextualize** — for each remaining lead, the full
   resolved account record.

Then it picks one of three campaign patterns (signal-led, pattern-led,
cold-start) and writes the sequence with opinionated rules baked in.

The frameworks below — three campaign shapes, Situation→Insight→Inquisition,
hard rules on length and CTAs — are borrowed from the
[Zevenue gtm-skills email-writer](https://github.com/Zevenue/gtm-skills/blob/main/.claude/skills/email-writer/skill.md).
What's new here is the pre-write Nous read.

## How to invoke

In Claude Code:

```
/cold-email-writer
```

— or *"write a campaign for these leads"*. The skill will ask for:

- The **lead list or segment** and where the leads came from (LinkedIn
  engagement / ICP search / hand-built / referral / inbound).
- **What you sell** in one line.
- The intended **cadence** — single email or a 3-step sequence.

## Setup

One env var:

```
export NOUS_API_KEY=pk_xxx
```

Or attach the Nous MCP connector to your Claude session. The skill uses
whichever is available.

## Core philosophy

**The prospect's situation comes first. Your product is not the hero.**
Open on their reality. Mention what you sell only when it's earned its
place.

**Targeting determines messaging.** A good signal lets the copy almost
write itself. If the email needs paragraphs, the targeting needs to be
sharper, not the prose.

**Ask for truth, not time.** Every email ends on a question the prospect
can answer in one line. Not *"do you have 15 minutes Thursday?"*. Try
*"is this still on your plate?"* / *"got this part wrong?"*.

**Learn from what's actually worked.** Last quarter's reply data beats
this quarter's hunch. The skill reads it; the skill respects it.

## The process

### 1. Suppress

Call the Nous `/v2/leads/check` endpoint (or the equivalent MCP `query`
tool) with the list. Drop anyone with status: `contacted`, `replied`,
`bounced`, `unsubscribed`, `dnc`. Report the drop count to the operator —
no surprises.

### 2. Learn from past campaigns

For the remaining leads' segment (LinkedIn engagement / ICP segment /
referral / inbound), query Nous for past campaign outcomes. Look at:

- Reply rate by tone (warm / blunt / founder-led)
- Reply rate by email length (short / standard)
- Reply rate by CTA style (truth-question / soft ask / direct calendar)
- DNC patterns (which template phrasings produced opt-outs)

If there's enough data for this segment (≥ 30 past sends with response
signal), the skill **uses the pattern that's winning** and tells the
operator which it picked and why. If not, the skill marks the run as
cold-start and tells the operator.

### 3. Contextualize each lead

For every remaining lead, call Nous `account` with their identifier.
Note the strongest recent signal — LinkedIn engagement, job change,
hiring signal, mentioned-on-podcast, recent product launch. This signal
is what email 1 opens on.

### 4. Choose the campaign pattern

Pick one of three:

#### Signal-led — when the lead has a clear public signal

Structure (Situation → Insight → Inquisition):
- **Line 1.** Describe the prospect's specific reality from the signal.
- **Line 2.** One insight that shows you've seen this pattern before.
- **Line 3.** Ask if you're right.

> *You just opened three SDR roles. Usually that means the team you have
> is buried in list-building instead of selling. Is that what's happening
> — or is it pure growth?*

#### Pattern-led — when past campaigns to this segment have a winning template

Use the angle, length, and CTA that produced the highest reply rate in
the past data. Personalize per-lead.

#### Cold-start — when there's no past pattern yet

Safe baseline: short, signal-grounded if there's any signal at all,
otherwise segment-fallback. One truth-question CTA. Three lines.

### 5. Write the sequence

Three emails per lead: initial touch + two follow-ups. Personalize from
the resolved record (first name, role, company, the signal). Hard rules
below.

### 6. QA

For each email: would a busy person reply? If not, rewrite. Run the hard-
rules checklist. Flag anything that breaks a rule for the operator.

### 7. Output

Format the campaign as a JSON array (one object per lead with `email1`,
`followup1`, `followup2`, `subject`, `classification_used`) so the
operator can pipe it into Lemlist, Smartlead, Instantly, or wherever.

## Hard rules

Borrowed from the Zevenue email-writer because they hold up:

- **Email 1**: under 75 words, 3 lines max, no link.
- **Follow-ups**: 4 lines max, under 60 words.
- **Subject lines**: 2–4 words, lowercase, no hype, no punctuation tricks.
- **Plain text only.** No HTML. No images.
- **One question per email.** Truth, not the calendar.
- **Never open on the product.** Open on the signal or their reality.
- **Forbidden phrases**:
  - "Hope this finds you well"
  - "Quick question"
  - "Just floating this"
  - "Circling back"
  - "Just bumping this"
  - "I wanted to reach out"

## Run on a recurring cadence

Pair this skill with the [`chase-non-responders`](../../routines/chase-non-responders.md)
routine and the [`creator-engagers`](../linkedin-engagers/SKILL.md) skill
for the full loop:

1. `creator-engagers` builds the list (sourcing).
2. `cold-email-writer` writes the campaign (drafting).
3. `chase-non-responders` keeps the follow-up alive on a 9am cron.

## Frequently asked questions

**What patterns does the skill actually learn from?**
Reply rate by tone, by length, by CTA style — segmented by signal source
and ICP. The skill queries your Nous workspace and reads what's working
in similar campaigns before drafting.

**What if I don't have past campaign data yet?**
Cold-start mode kicks in. The skill uses a safe baseline (short,
signal-grounded if possible, one truth-question CTA) and starts building
the pattern from your first reply set. The next campaign on this segment
gets sharper automatically.

**Does it replace my sequencer (Lemlist, Smartlead)?**
No. The skill writes the campaign; you push the output into your
sequencer. Or wire the `chase-non-responders` routine to push the chase
step on top.

**Can I edit the hard rules?**
Yes — they're at the bottom of this file. Edit your installed
`~/.claude/skills/cold-email-writer/SKILL.md` to change them per
deployment. The skill obeys the file at run time.

**Why borrow from Zevenue's email-writer?**
The 3-pattern structure (situation → insight → inquisition) and the
hard rules on length and CTAs hold up — they're hard-won opinions from
a working agency. There's no value in reinventing them. What we add is
the Nous-specific pre-write read: suppression and pattern lookup. Same
opinions, plus your workspace's own evidence layered on top.
