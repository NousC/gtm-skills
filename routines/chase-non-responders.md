# Chase Non-Responders — Daily Routine

A Claude Code routine that finds leads who went quiet and drafts the
follow-up using their full Nous account record. Runs every morning at 9am.
Drafts land in your outbound tool, or back in Nous for review.

> **Why this exists.** Half of outbound is the chase, but the chase is what
> every rep skips. This routine does it the way you would if you had time:
> every day, only on leads who actually went quiet, using the full context
> Nous holds about them — not "just floating this up to the top of your inbox".

## What you need

- **Claude Code on Pro / Max / Team / Enterprise** with Claude Code on the
  web enabled. Cloud routines need this. (Free tier alternative: set this up
  as a local Desktop scheduled task on your machine — same prompt, runs from
  your laptop instead of Anthropic's cloud.)
- The **Nous MCP connector** attached to your Claude account:
  `https://claude.ai/customize/connectors`.
- *(Optional)* A Lemlist / Smartlead / Instantly MCP connector if you want
  drafts pushed straight into a campaign step. Without one, drafts land back
  in Nous as `chase_draft` note observations for human review.

## Setup — from the CLI

```bash
/schedule daily chase non-responders at 9am
```

Claude walks you through the same form below. Paste the prompt when prompted
for instructions.

## Setup — from the web

1. Open https://claude.ai/code/routines → click **New routine**.
2. **Name:** `Chase non-responders`.
3. **Instructions:** paste the [prompt](#the-prompt) below.
4. **Connectors:** attach **Nous** (required). Add your outbound connector
   if you have one. Remove the rest.
5. **Select a trigger:** *Schedule* → *Daily* → 9:00 AM (your local time).
6. Click **Create**.

It fires the next morning. To run it once now, click **Run now** on the
routine's detail page.

## The prompt

```
Daily follow-up on non-responders.

You have the Nous MCP attached. Use it to do the following:

1. Identify the leads to chase. Call the Nous `query` tool with:
   - scope: any lead list in the workspace
   - filter: created in the last 30 days
   - filter: has at least one outbound interaction observation
     (`interaction.email_sent` or outgoing `interaction.linkedin_message`)
   - filter: no inbound reply observation
     (`interaction.email_reply` or incoming `interaction.linkedin_message`)
     in the last 5 days
   - exclude: any lead with a `chase_draft` note observation written
     in the last 7 days

2. Cap the result at 25 leads. If more match, take the most recently engaged.

3. For each lead, call the Nous `account` tool with their identifier (email
   or linkedin_url) to get the full resolved record.

4. Read each account's timeline. Pick the chase channel:
   - If the lead has a public engagement observation in the last 14 days
     (LinkedIn comment / reaction, a website revisit, a signal_ingest event),
     follow up on LinkedIn.
   - Otherwise, follow up by email.

5. Draft ONE follow-up message per lead:
   - Under 60 words.
   - Open on something NEW — a recent signal, a recent post they engaged on,
     a job change, a hiring signal on their company. Never "I'm just
     following up".
   - End with a question they can answer in one line ("Is this still on your
     plate?", "Is the X problem still real for you?").
   - Forbidden: "Hope this finds you well", "Quick question", "Circling back",
     "Just bumping this up your inbox".

6. Deliver the draft:
   - If an outbound connector (Lemlist / Smartlead / Instantly) is attached
     AND a campaign named "Chase" exists, push the draft as a new step for
     that lead in that campaign.
   - Otherwise, write the draft back to Nous as a note observation on the
     lead: `property: chase_draft`, `value: { channel, body, reason }`.

7. End the run with a one-line summary per chase, plus totals:
   - N leads identified
   - M chases drafted (E email, L LinkedIn)
   - K pushed to outbound, J written back to Nous for review
   - S skipped, with the top 3 skip reasons.
```

## Tuning knobs

- **The non-responder window** is `5 days` by default. Change the number in
  step 1 before saving the routine.
- **The per-run cap** is `25 leads`. Raise it if you have the bandwidth to
  review; lower it to keep drafts manageable.
- **The chase cool-down** is `7 days` — same lead won't be chased twice in
  a week. Adjust the exclusion filter in step 1.

## What "drafts in Nous" looks like

If you don't have an outbound connector attached, the routine writes each
draft back to Nous as a `chase_draft` note observation on the lead's record.
Review them on the lead's detail page in the People view — they show up
under the Notes / Facts tab today, and under the dedicated Public Signals
tab when that ships.
