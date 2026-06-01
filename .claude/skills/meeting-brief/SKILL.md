---
name: meeting-brief
description: Before a meeting, pull everything Nous knows about the person, scrape their last four weeks of LinkedIn posts via Apify for themes, opinions and voice, profile their company from its website, read your own GTM profile from Nous, then write a sharp pre-meeting brief and save it back onto the account record. Use it 1:1 before a call or first conversation. Not for bulk lists.
---

# Pre-meeting brief

## What it does

You point it at a meeting, a person, or a company. It gathers four sources —
**what Nous already knows**, **their recent LinkedIn posts**, **their company
website**, and **your own GTM profile** — and writes you a brief you can read in
two minutes and walk in sharp: who they are, where the relationship stands,
what is on their mind, the language they use, the angle for your offer, and the
questions to ask. Then it saves the brief back onto the account in Nous so it is
on the record next time.

This is a deliberate, 1:1 move before a real conversation. It is not a
list-builder, and it is not meant to run at scale.

The flow: **Nous → LinkedIn → Apify → website → Claude Code → Nous.**

## How to invoke

`/meeting-brief` — or *"brief me on my 2pm with Sarah Chen at Acme."*

The skill will resolve what it needs:

- **Who** — a name + company, an email, or a LinkedIn URL.
- **The meeting** — if a calendar connector is attached and you reference a time
  ("my 2pm"), it reads the event and briefs every external attendee.
- **Look-back window** for their posts (default `28` days, four weeks).

## First-run setup (you, the agent, run this once as a short interview)

The first time this runs, check what's connected and fill only the gaps. Detect
before you ask, ask one thing at a time, and do the work for the user wherever
you can. Once everything is in place, skip straight to the brief on every later
run and only re-ask if something's missing.

**1. Nous — the account record this brief reads and writes back.**
Check whether the Nous MCP is connected by trying the `get_account` tool.
- Connected → move on.
- Not connected → set it up for them. Tell them exactly this, don't make them
  hunt:
  > "This brief runs on Nous. Connect it in one line:
  > `claude mcp add nous -e NOUS_API_KEY=<your-key> -- npx -y @opennous/mcp`
  > Get your key at opennous.cloud → Settings → API keys (the `pk_` one). No
  > account yet? It's free at opennous.cloud — sign up, grab the key, paste it
  > in. Want me to wait while you do that?"
  After they run it, confirm `get_account` works before continuing.

**2. Apify — the LinkedIn post scrape.**
Check for `APIFY_TOKEN` in the environment.
- Present → move on.
- Missing → "I read their recent posts through Apify, about $0.30 per person.
  Add your token once: `export APIFY_TOKEN=apify_api_xxx` (apify.com → Settings
  → Integrations)."

**3. Calendar (optional).**
Only if they want a whole meeting briefed: "Attach a Google Calendar connector
and I'll brief every external attendee. Otherwise just name the person." A
Google Docs / Drive connector is optional too, to drop the brief into a doc.

When the inputs are ready, confirm before spending: "Ready. Brief <person>? One
Apify call, about $0.30." Then run the brief.

The post scrape uses one Apify HarvestAPI actor (your token must have access):

| Actor (slash form) | Used for |
|---|---|
| `harvestapi/linkedin-post-search` | the person's own recent posts |

In Apify URLs the slash becomes `~` (`harvestapi~linkedin-post-search`).

## Core philosophy

**Walk in knowing the person, not just the company.** The brief is about who
they are and what they care about right now, grounded in real signals.

**Every line is sourced.** Each point traces to where it came from — the Nous
timeline, a specific post, or the company site. Never invent a fact, an opinion,
or a quote. If a source is thin, say so.

**Read their voice so you can meet it.** Pull the actual language they use, so
your message sounds like a peer, not a pitch.

**Connect their world to your offer.** The angle comes from your GTM profile
crossed with what they actually care about, not a generic value prop.

**The brief is a record, not a throwaway.** It is written back to Nous so the
next person to touch this account starts ahead.

## The process

Claude runs these steps in order. If a source is missing, note it in the brief
and continue — never fabricate to fill a gap.

### 1. Resolve the meeting and the person

Parse the prompt for the person and company. If a calendar connector is attached
and the operator referenced a time, read the event: capture the attendees, the
agenda, and the organizer. Brief each **external** attendee (skip people on your
own domain). Otherwise brief the single person named.

Normalize each LinkedIn URL before any lookup or scrape:

```js
function normalizeLinkedInUrl(u) {
  return u.toLowerCase()
          .replace(/^https?:\/\//, "https://")
          .replace(/\/\/www\./, "//")
          .replace(/[?#].*$/, "")
          .replace(/\/+$/, "");
}
```

### 2. Pull what Nous already knows

Call the Nous `get_account` MCP tool with the person's email or LinkedIn URL
(fall back to name + company). From the resolved record, capture:

- identity — role, seniority, company, location
- the timeline — past emails, calls, meetings, and signals, newest first
- current pipeline stage and how long they have been in it
- open commitments or action items from the last conversation
- the ICP fit score, if Nous has scored them

If Nous does not know this person, record that explicitly and continue on public
data only. Do not block.

### 3. Read their recent LinkedIn posts (Apify)

Scrape their own posts from the last `WINDOW_DAYS` (default 28).

```bash
curl -s -X POST \
  "https://api.apify.com/v2/acts/harvestapi~linkedin-post-search/run-sync-get-dataset-items?token=$APIFY_TOKEN&timeout=120" \
  -H "Content-Type: application/json" \
  -d '{ "profileUrls": ["<PERSON_PROFILE_URL>"], "maxItems": 20, "sortBy": "date" }'
```

`sortBy: "date"` is required. Drop posts older than `WINDOW_DAYS`. From what
remains, have Claude extract:

- **Themes** — the 3-5 topics they keep returning to.
- **Opinions** — stances they have taken, things they champion or push back on.
- **Energy** — what they are excited about, frustrated by, or recently launched.
- **Voice** — their tone, plus 2-3 short verbatim phrases that sound like them.

Quote only what is actually in a post. If they have not posted in the window,
say so and lean on the other sources.

### 4. Profile their company from the website

Resolve the company domain from the Nous record or the person's profile. Plain
`fetch()` the homepage, and `/about` and `/product` if they exist. Strip to text
and have Claude write a short profile:

- what the company actually does, in one or two sentences
- who they sell to
- their positioning and any recent news visible on the site

No scraping infrastructure — a plain fetch and a read. If the site is
JavaScript-heavy and returns little, note that the profile is thin.

### 5. Pull your GTM angle from Nous

Call `get_gtm_profile` to read what you sell, your ICP, and your value props.
This is the lens for the angle: cross what they care about (steps 3-4) with what
you offer, and find the specific wedge for this account. If no GTM profile is
set, write the angle from the conversation context instead and suggest setting
one up.

### 6. Write the brief

Synthesize everything into this structure. Keep it tight — two minutes to read.
Every line cites its source in brackets, e.g. `[timeline]`, `[post 2026-05-21]`,
`[site]`.

```
SNAPSHOT
  Who, role, company one-liner. Relationship status: stage, last touch,
  days since, ICP fit.

WHERE THINGS STAND
  Open threads, commitments, the last thing said. [timeline]

WHAT'S ON THEIR MIND
  Themes and opinions from their posts; recent company moves. [post …] [site]

THEIR VOICE
  Tone, plus 2-3 phrases they actually use. Mirror this. [post …]

YOUR ANGLE
  How your offer maps to their world. The specific wedge for this account.
  [gtm_profile]

BRING THIS UP
  3-5 talking points or questions, each grounded in a real signal above.

WATCH-OUTS
  Sensitivities, unresolved items, anything to avoid. [timeline]

NEXT STEP
  The one move this meeting should produce.
```

### 7. Write the brief back to Nous

Save the brief onto the account with the `record` MCP tool, as a note
observation:

- `property: note.meeting_brief`
- `value: { meeting, date, brief, sources }`
- `external_id: "meeting_brief:<person_url>:<meeting_date>"` — idempotent, so
  re-running for the same meeting updates rather than duplicates.

### 8. Deliver

Print the brief in chat. If a Google Docs / Drive connector is attached and the
operator asked for it, also write the brief to a doc and return the link.

## Hard rules — never break these

- **Never invent.** No fact, opinion, quote, or company detail that is not in a
  source. Thin source → say it is thin.
- **Cite every line.** The reader has to know whether a point came from the
  record, a post, or the site.
- **External attendees only.** When briefing a meeting, skip people on your own
  domain.
- **One person, one paid Apify run.** Do not loop the post scrape; `maxItems: 20`
  is enough for a four-week window.
- **1:1 only.** If asked to brief a long list, stop and point the operator at the
  `linkedin-engagers` skill — this one is for a real, upcoming conversation.
- **Write it back.** The brief is not done until it is on the Nous record.

## Brief a whole day (optional)

Once it works 1:1, run it each morning over today's external meetings:

```bash
# 7:30am — brief every external meeting on today's calendar
30 7 * * *  cd ~/work && claude -p "/meeting-brief brief every external meeting on my calendar today"
```

Needs the calendar connector attached. Each brief is still written back to its
own account in Nous.

## Frequently asked questions

**What if Nous has never seen this person?**
The brief still runs on public data — their posts and their company site — and
notes that there is no prior history. The moment you log the meeting, the next
brief starts from a record.

**Does it score them against my ICP?**
It reports the ICP fit if Nous has scored them, and the angle is drawn from your
GTM profile. Set up your GTM profile at opennous.cloud → GTM Context for the
sharpest angle.

**Can it brief more than one person?**
Yes, for the attendees of one meeting. It is not built to brief a list of
prospects — use `linkedin-engagers` to build and score a list.

**Where does the brief live afterwards?**
On the account in Nous as a `note.meeting_brief` observation, and in chat. Add a
Google Docs connector to also drop it into a doc.
