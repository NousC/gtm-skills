---
name: campaign-writer
description: Write a full outbound campaign — a multi-touch sequence for a segment — grounded in your Nous GTM profile, your ICP, and what has actually replied in your past campaigns to that segment. It learns the winning angle from real reply data, suppresses anyone already contacted, writes the sequence in the pattern that converts, and records the copy back so the next campaign learns from this one. Push to your sequencer or keep it as a draft.
---

# Campaign Writer

## What it does

You give it a segment and a goal. It writes the whole campaign — a multi-touch
sequence — not from a template, but from **what has actually replied** in your
own past outbound to that kind of buyer. It reads your GTM profile, queries Nous
for the variants and segments that earned positive replies, picks the angle from
that evidence, suppresses anyone you've already touched, and writes the sequence
in the winning pattern. Then it records the copy back to Nous so when replies
land, the next campaign is even sharper.

The flow: **Nous → Claude Code → your sequencer.**

## How to invoke

`/campaign-writer` — or *"write a 4-email sequence for my RevOps at Series B
SaaS segment, in the angle that's been replying."*

It resolves what it needs:

- **Segment** — a Nous lead list, a query, or a plain description.
- **Channel** — email, LinkedIn, or both (default email).
- **Goal** — book a meeting, earn a reply, drive to a resource.
- **Length** — touches in the sequence (default 3).

## First-run setup (you, the agent, run this once as a short interview)

Detect what's connected, ask only for what's missing, one thing at a time.

**1. Nous — your GTM profile and what's replied.** Check whether the Nous MCP is
connected by trying `get_gtm_profile`.
- Connected → continue.
- Not → "This skill runs on Nous. Connect it in one line:
  `claude mcp add nous -e NOUS_API_KEY=<your-key> -- npx -y @opennous/mcp`. Get
  your key at opennous.cloud → Settings → API keys (the `pk_` one). No account?
  It's free at opennous.cloud." Confirm `get_gtm_profile` returns before moving on.

**2. Is your GTM profile set?** The angle and the suppression both read from it.
If `get_gtm_profile` comes back empty: "Set up your GTM profile at
opennous.cloud → GTM Context first, so the angle is drawn from what you actually
sell, not a guess."

**3. Which sequencer do you send from?** Ask: "Smartlead, Instantly, or
HeyReach? Add that key and I'll push the finished campaign in; otherwise I'll
leave it as a draft." Also note `NOUS_API_KEY` lets it record the copy per
variant (`/api/campaign-messages`) so the next campaign learns.

Then confirm the segment, channel, and length before drafting.

## Core philosophy

**Write from evidence, not templates.** The campaign mirrors the tone, length,
and angle that actually earned replies for this segment.

**Ground it in your GTM.** The angle comes from your GTM profile crossed with
the segment's pain, never a generic value prop.

**Suppress before you write.** Never draft to someone you've already burned.

**A campaign is a hypothesis.** Record the copy so its results sharpen the next
one. The loop only closes if the copy is on the record.

**Human approves before it sends.** It drafts and proposes; you ship.

## The process

### 1. Define the campaign

Resolve the segment, channel, goal, and length from the prompt. If the segment
is a description rather than a list, translate it into a query scope.

### 2. Read your GTM profile

Call `get_gtm_profile` (or `GET /v2/workspace/facts?categories=ICP,Market,Product,Pricing,Competitors`).
Use the `ICP`, `Product`, and `Competitors` facts as the raw material for the
angle and the proof points.

### 3. Learn what has worked

This is the edge. Call the Nous `query` tool for the positive replies and read
their attribution + firmographics:

```
query {
  scope:  { property: 'interaction.positive_reply', since_days: 90 },
  return: 'entities'
}
```

Each row now carries `most_recent_attribution` (the campaign / step / **variant**
that earned the reply) and `firmographics` (industry, title, company size).
Group them: which **variant** converted, and for which **industry / title**.
Then read the exact copy of the winning variants:

```
GET /api/campaign-messages?campaignId=<id>      # subject + body per variant
```

And pull the semantic themes across the replies:

```
query {
  question: 'what do these positive replies have in common',
  scope:    { property: 'interaction.email_received', since_days: 90 }
}
```

If there's no reply history for this segment, say so and write a safe baseline
(short, signal-grounded, one truth-question CTA). The loop sharpens from the
first reply set.

### 4. Clean and scope the list

Suppress anyone already in motion — query the workspace for prior outbound or
replies on the target identifiers and drop them:

```
query { scope: { property: 'interaction.email_sent' }, return: 'entities' }
```

Drop bounced / unsubscribed / already-replied. Capture the segment's shared
firmographics and the pain they imply.

### 5. Pick the angle

Cross your offer (step 2) × the segment's pain × what converted (step 3).
Propose 2-3 angles, recommend the one the evidence supports, and let the
operator confirm before drafting.

### 6. Write the sequence

Write the touches in the winning pattern:

- **Touch 1** opens on the segment's real trigger + the chosen angle.
- **Touches 2..N** each add a new reason or proof, never "just bumping".
- Rules: under 75 words, one CTA, ask for truth not just the calendar, no
  "hope this finds you well".
- Variables for per-lead personalization: `{{first_name}}`, `{{company}}`,
  `{{role}}`, and a `{{signal}}` line.
- A subject line per email.

### 7. Review, ship, and record

Present the full campaign (angle rationale, the evidence it drew from, the
suppression summary, the sequence). On approval:

- **Push** to the sequencer (Smartlead / Instantly / HeyReach) with the cleaned
  leads + variables, or keep it as a draft.
- **Record the copy** so replies are attributable when they come back:
  ```
  POST /api/campaign-messages
    { campaignId, variant, subject, body, source: 'campaign_writer' }
  ```
  This is what closes the loop — the next run of step 3 will see how this
  campaign performed.

## Hard rules — never break these

- **Write from evidence.** Cite which past variants / replies informed the
  angle. If there's no history, label it cold-start.
- **Suppress first.** No draft to anyone already contacted, replied, bounced,
  or unsubscribed.
- **Human approves before send.** Never push to a live sequencer unprompted.
- **Record the copy.** The campaign isn't done until its copy is on the record,
  keyed by variant, so the loop can learn.

## Frequently asked questions

**What if I have no past campaign data for this segment?**
Cold-start mode. It writes a safe baseline and flags it. The first reply set
makes the next campaign on this segment sharper, automatically.

**How does it know which email worked?**
Nous records the campaign / step / variant on every send and reply, so `query`
can group positive replies by the exact email that earned them, and
`/api/campaign-messages` holds that email's copy.

**Does it send on its own?**
No. It drafts and proposes; you approve, then it pushes to your sequencer or
leaves a draft.

**Does it replace my sequencer?**
No. It writes the campaign and can push it in; Smartlead / Instantly / HeyReach
still do the sending.
