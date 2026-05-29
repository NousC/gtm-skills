# Cold Reply Agent — Routine

An API-triggered Claude Code routine. When a cold-email reply lands in
Instantly or Lemlist, the sequencer POSTs to the routine's URL — the routine
classifies the response, picks the matching template from your repo, and
sends the right next message through Gmail (or your sequencer).

> **Why this exists.** Cold email converts on the *first* contextual response.
> The "yes, tell me more" deserves a calendar link in minutes. The polite
> "not now" deserves a calm acknowledgement and a nurture flag. The hostile
> reply should never get touched. Doing this by hand stops scaling past the
> second client campaign.

## What you need

- **Claude Code Pro / Max / Team / Enterprise** (cloud routines require it).
- A **GitHub repo** with two context files (see below). The routine clones
  this repo on every fire.
- **Nous MCP connector** on your Claude account
  (`claude.ai/customize/connectors`) — for the live account lookup.
- **Gmail MCP connector** — for the actual send. Or your sequencer's
  connector if it has one (Lemlist, Smartlead, Instantly).

## Setup

API triggers cannot be created from the CLI. Use the web.

1. Open `https://claude.ai/code/routines` → **New routine**.
2. **Name:** `Cold reply agent`.
3. **Instructions:** paste the [prompt](#the-prompt) below.
4. **Repositories:** add the repo holding your `context/` files.
5. **Connectors:** attach **Nous** + **Gmail** (or your sequencer). Remove
   the rest.
6. **Triggers:** choose **API** → save the routine → click **Generate token**
   and copy it immediately (it's shown once).
7. **Copy the URL.** It looks like
   `https://api.anthropic.com/v1/claude_code/routines/<id>/fire`.
8. **Wire the webhook.** In Instantly or Lemlist, set the *"reply received"*
   webhook to POST to that URL with:
   - `Authorization: Bearer <token>`
   - `anthropic-beta: experimental-cc-routine-2026-04-01`
   - `anthropic-version: 2023-06-01`
   - `Content-Type: application/json`
   - Body: `{ "text": "<the reply payload, as a JSON string>" }`

Some sequencers can't post the payload nested under `text` — wrap their
webhook through a relay (an `n8n` step, a Cloudflare Worker, a Make
scenario) that re-shapes the body to `{ "text": "..." }`.

## The prompt

```
You are the cold-reply agent. An Instantly or Lemlist webhook just fired
with a reply payload in the `text` field. Your job: classify, draft, and
send the right next message.

1. PARSE the webhook payload from `text`. Extract:
   - the lead's email
   - the original campaign / sequence name
   - the reply body (and subject, if present)
   - any sentiment score the sender's tool included

2. LOAD THE CONTEXT. From the cloned repo, read:
   - context/company.md — what you sell, calendar link, value props, proof
   - context/reply-templates.md — the response template per classification

3. LOOK UP THE LEAD. Call the Nous `account` tool with the lead's email.
   Read their full record: prior outbound, their company, their role, any
   recent signals (LinkedIn engagement, job changes, etc.).

4. CLASSIFY the reply into exactly ONE of:
   - positive_interested   — "yes, tell me more" / "send me a time"
   - positive_referral     — "I'm not the right person, talk to X"
   - neutral_info_request  — "what does it do?" / "send more info"
   - neutral_timing        — "not now, maybe Q3" / "circle back later"
   - negative_soft_no      — "not interested" (polite)
   - negative_hard_no      — "stop emailing" / "unsubscribe" / hostile
   - autoreply_ooo         — out-of-office autoreply
   - unclear               — can't confidently classify

   If confidence < 80%, classify as `unclear`.

5. APPLY HARD RULES:
   - `negative_hard_no` → write a `dnc` note observation back to Nous
     via the Nous MCP. DO NOT send any reply.
   - `negative_soft_no` → send ONE polite acknowledgement using the
     matching template. Mark the lead as `cool` in Nous.
   - `autoreply_ooo` → write a note observation in Nous. DO NOT reply.
   - `unclear` → draft a response, write it back to Nous as a
     `draft_reply` note for human review. DO NOT send.

6. DRAFT THE RESPONSE. Use the matching section from
   context/reply-templates.md. Personalize with the lead's first name,
   role, and company. Never invent claims about the company — only what's
   in context/company.md. If the template references the calendar link,
   use the one in context/company.md.

7. SEND. Use the Gmail MCP (or the sequencer connector if attached) to
   reply on the existing thread. Recipient = the lead's email. Do not
   CC anyone unless the original thread did. Subject line = "Re: " +
   the original subject.

8. LOG the sent message back to Nous as an `interaction.email_sent`
   observation on the lead, with:
   - `value.body` = the message body you sent
   - `value.classification` = the classification you chose
   - `value.template_used` = the template section name
   - `external_id` = "cold_reply_agent:" + the routine run id
   The external_id makes the run idempotent — if the webhook fires twice
   on the same reply, no double-send.

9. END with a one-line summary:
   "<classification> · <action: sent / drafted / logged_only / dnc> ·
    <lead email> · <run id>"
```

## Context files

These two files live in the cloned repo. The routine reads them on every
fire, so changes take effect on the next reply without re-deploying.

### `context/company.md` (template)

```markdown
# About [Company]

## What we sell
[One paragraph: what the product does, who it's for, the problem it solves.]

## Value prop in one line
[The one line that goes in cold replies. Keep it sharp.]

## Booking link
https://cal.com/yourname/intro

## Key proof points
- [Customer A — concrete outcome]
- [Customer B — concrete outcome]
- [Numbers worth quoting — funding, customers, scale]

## Tone of voice
Short. Specific. No jargon. One question per email.
Never use "Hope this finds you well", "Quick question", "Just floating this".
```

### `context/reply-templates.md` (template)

```markdown
# Reply templates per classification

## positive_interested
Triggers on: "yes tell me more", "send me a time", "interested", "open to chat".

Response:
> Thanks [first_name] — happy to walk through it. Here's my calendar:
> [CALENDAR_LINK]. Pick any time that works.

Rules:
- Always include the calendar link from context/company.md.
- One line max. No paragraph.

## positive_referral
Triggers on: "I'm not the right person", "talk to X", "redirect to ...".

Response:
> Appreciate the redirect, [first_name]. I'll reach out to [referred_name] —
> anything I should know that would help them?

## neutral_info_request
Triggers on: "what does it do?", "send more info", "tell me more about it".

Response:
> Quick version, [first_name]: [VALUE_PROP_ONE_LINE]. Two ways we could go —
> a 20-minute walk-through ([CALENDAR_LINK]) or I send a one-pager. Which is
> easier?

## neutral_timing
Triggers on: "not now", "maybe Q3", "circle back later".

Response:
> Appreciate the honesty, [first_name]. I'll circle back in [PERIOD]. If
> anything changes in the meantime: [CALENDAR_LINK].

## negative_soft_no
Triggers on: "not interested", "we're good", "all set".

Response:
> Got it, [first_name] — thanks for the quick reply. I'll close the loop
> on my end.

## negative_hard_no
Triggers on: "stop emailing", "unsubscribe", hostile language.

**DO NOT REPLY.** Mark DNC in Nous.

## autoreply_ooo
Triggers on: classic OOO patterns, "I'm out of the office until …".

**DO NOT REPLY.** Log only.

## unclear
Anything that doesn't classify cleanly above.

**DO NOT SEND.** Draft a response and write back to Nous as a
`draft_reply` note for human review.
```

## Hard rules baked in

- Confidence threshold: classifications below 80% → `unclear` → human review.
- Hostile replies are never auto-answered.
- OOO autoreplies are never auto-answered.
- One reply per routine fire; the `external_id` prevents double-sends if the
  webhook re-fires.
- Every send is logged back to Nous as an observation — so the lead's full
  history stays current.

## Tuning knobs

- **Confidence threshold** — 80% by default. Raise it (e.g. 90%) if you want
  more replies to route to human review. Lower it (e.g. 70%) once you trust
  the templates.
- **Classification set** — edit the eight categories in the prompt to add
  domain-specific ones (e.g. `pricing_question`, `competitor_mention`). Add
  a matching template section.
- **Send connector** — switch from Gmail to your sequencer by removing the
  Gmail connector and attaching Lemlist / Smartlead / Instantly. The prompt
  doesn't need to change.

## Roadmap

- A `pricing_question` classification that pulls from a `pricing.md` context
  file. Most cold replies that look like info-requests are actually
  pricing-fishing; routing them specifically helps.
- An `objection_handling` set of templates keyed to common objection
  patterns (timing / budget / authority / need).
- Auto-add to a CRM "Working" stage when classification is
  `positive_interested` and the calendar link gets sent.
