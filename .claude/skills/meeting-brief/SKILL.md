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

- **identity** — role, seniority, **the company they work at** (name + one-liner
  on what it does), location.
- **how you came into contact** — read the timeline/source to name the channel
  and origin: "you've been talking on LinkedIn", "a Gmail thread", "they replied
  to an Instantly/HeyReach campaign", "they booked via Cal.com", "inbound off a
  comment on your post". This is the relationship's *entry point* — always state
  it.
- **call number** — is this the **first** conversation, a **second**, or an
  **ongoing** relationship? Infer from prior meetings/calls on the timeline and
  from any earlier meeting briefs (see below).
- **the timeline** — past emails, calls, meetings, and signals, newest first.
- **tech stack / platforms they use** — pull any tools mentioned in the record
  (CRM, Clay, n8n, outbound platforms, etc.), so the angle can speak to it.
- current pipeline stage and how long they have been in it.
- open commitments or action items from the last conversation.
- the ICP fit score, if Nous has scored them.

**Retrieve any prior meeting briefs.** Prior briefs come back **on the
`get_account` call you already made** — scan its facts for `note.*` claims whose
metadata `doc_type` (or category) is `meeting_brief`; the newest is the last
brief, full text included. Read those directly — this is the reliable path.
(`search_notes` is a semantic, embeddings-backed search and can miss a brief
saved moments ago, so don't depend on it for freshly-written briefs; use it only
as a secondary, cross-meeting lookup.) If a prior brief exists: this is **not** a
first call — summarise what was last discussed and the previous NEXT STEP, and
write *this* brief as a continuation ("last time you covered X; since then …").
This is how the brief compounds across meetings instead of starting cold.

If Nous does not know this person, record that explicitly and continue on public
data only. Do not block.

### 3. Read their recent LinkedIn posts (Apify)

**First, get a *scrapeable* profile URL — this is the #1 failure point.**

The Apify post-search actor only accepts a **public vanity URL**
(`linkedin.com/in/aakash-sethi`). It returns **0 posts** for a **member-URN
URL** (`linkedin.com/in/acoaa…` / `/in/ACoAA…`) — LinkedIn's internal id, which
is what contacts first met via a **LinkedIn DM/invite** carry. So:

1. Pull the URL from the Nous record, in this order: the `linkedin_url` fact,
   then `channels.linkedin.url`. (Nous now resolves and stores a real vanity
   handle for DM-sourced contacts on each LinkedIn sync, so most records will
   already have a good one.)
2. **Check the slug.** If it matches `/in/acoaa…` (case-insensitive), it is a
   member URN — **do not scrape it, it will return 0**. Treat as "no URL yet."
3. **If no scrapeable URL:** try, in order — (a) ask the operator to run a
   LinkedIn sync in Nous, which back-resolves the handle, then re-read the
   record; (b) `WebSearch` `"<name>" <company> linkedin` and take a confident
   `/in/<vanity>` match; (c) if still nothing, **skip the scrape**, say posts
   were unavailable and why, and lean on the company site + timeline. Never
   guess a slug, and never scrape a member-URN URL "just in case."

Then scrape their own posts from the last `WINDOW_DAYS` (default 28):

```bash
curl -s -X POST \
  "https://api.apify.com/v2/acts/harvestapi~linkedin-post-search/run-sync-get-dataset-items?token=$APIFY_TOKEN&timeout=120" \
  -H "Content-Type: application/json" \
  -d '{ "profileUrls": ["<VANITY_PROFILE_URL>"], "maxItems": 20, "sortBy": "date" }'
```

`sortBy: "date"` is required. **Sanity-check the result:** 0 posts for someone
the record shows posts/activity for usually means a bad (member-URN) URL, not an
inactive poster — re-resolve via step 3 before concluding they are quiet. Drop
posts older than `WINDOW_DAYS`.

**Keep each post's `linkedinUrl`** — the scraper returns it on every post (e.g.
`linkedin.com/posts/<slug>_…-activity-<id>`). Every post you cite in the brief
must be a clickable markdown link to that URL, so the operator can open the
actual post. Capture the `postedAt.date` too, for the citation date.

Then **audit the posts properly** — this is not a one-liner. Read everything in
the window and build a real picture:

- **Themes (audit, not a list)** — the topics they keep returning to, each as
  *2-3 sentences*: what the theme is, what they actually said about it, and a
  recent example post. The reader should finish this and genuinely understand
  what this person is thinking about right now and how they think about it.
- **What they're doing** — concretely: what are they building, launching,
  hiring for, attending (events/hackathons), shipping? What do they spend their
  days on, as evidenced by the posts?
- **Tools & platforms they name** — every product/tech they mention (Clay, n8n,
  HubSpot, Claude, ChatGPT, Profound, etc.). This is gold for finding overlap
  with what you do.
- **Opinions & stances** — what they champion, push back on, or have strong
  views about.
- **Voice (a real profile, several sentences)** — characterise how they post:
  the form they use (teardown, build-log, hot take, story), what they focus on,
  the platforms/tools they reference, what they share about their day, and the
  tone. Then state, in a sentence, what *their voice is* so you can meet it.
  Back it with **3-5 short verbatim phrases** that sound like them.

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

### 5. Pull your GTM angle from your own ICP file

The ICP now lives in the user's workspace, not a web form. **Read their ICP file**
(`context/icp.md` first; else `.claude/`, `gtm/`, `icp*`/`positioning*`) for what
you sell, your ICP, and your value props — the `<!-- nous:icp -->` block holds the
learned model of which signals predict a win. Call `get_icp_model` to refresh it
(`get_icp` re-syncs if the file changed); `get_gtm_profile` fills any gaps not in
the file. This is the lens for the angle: cross what they care about (steps 3-4)
with what you offer, and find the specific wedge for this account. If there's no
ICP file yet, write the angle from the conversation context instead and offer to
create `context/icp.md` so the ICP lives in their repo going forward.

### 6. Write the brief

Synthesize everything into this structure — the Nous document house style:
a `#` title, a `>` lede that IS the read, a plain `Key: value` snapshot block, a
`---`, then `## Title-case` sections. The middle sections (What's on their mind,
Their voice) earn real depth — the rest stays tight. Every line cites its source
in brackets, e.g. `[timeline]`, `[post 2026-05-21]`, `[site]`, `[prior brief]`.

```markdown
# Pre-meeting brief — <Person>, <Company>

> The read, up top — 3-5 sentences, written like you're briefing a colleague in
> the hallway 60 seconds before the call. Who this person is, where they're at,
> why they matter to you, how warm the relationship is, what kind of conversation
> to expect, and the one thing to walk away with. The synthesis that ties the
> whole brief together, not a restatement of the bullets below.

Person: <name> — <role> at <Company> (one line on what the company does)
Meeting: <meeting name>, <YYYY-MM-DD>
Call: first call / second call / ongoing relationship
How you met: LinkedIn / Gmail thread / replied to a campaign / booked via Cal.com / inbound off your post
Stack: <tools/platforms they work with, if known>
ICP fit: <score + label, if Nous has scored them>
Sources: [timeline] [post …] [site] [prior brief]

(Skip CRM-plumbing noise like "connected", "awaiting reply", "last message".)

---

## Where things stand
The relationship in 2-4 lines: how it started, what's happened since, the last
thing said, any open thread or commitment. If a prior brief exists, lead with
what changed since then. [timeline] [prior brief]

## What's on their mind
The longest section — the audit, give it real room. A genuine read on what
they're thinking about, from their posts and company moves. For each recurring
theme, write 3-5 sentences: what the theme is, what they actually said
(paraphrase + a short quote), a concrete example, and why it matters for the
conversation. Then cover what they're building/shipping/attending right now and
the tools they keep naming. Every post referenced is a clickable markdown link to
its `linkedinUrl`, e.g. "[their 06-14 hackathon post](https://www.linkedin.com/posts/…)".
The reader should finish this section genuinely understanding this person's
current world. Err long here — this is the part the operator reads twice. [post …] [site]

## Their voice
A profile, not a label. How they show up: the form they post in, what they focus
on, the platforms they reference, what they reveal about their day, and their
tone — then one line naming what their voice *is* so you can mirror it. Back it
with 3-5 short verbatim phrases (linked to the posts they came from). [post …]

## Your angle
How your offer maps to their world. The specific wedge for this account. [gtm_profile]

## Bring this up
Fully-formed questions, ready to say out loud — 3-5 of them. Each one names HOW
you know it, then asks the actual question, in words you could say verbatim. Each
must trace to a real signal above.
- "I saw you posted about <thing> — <specific question>?"
- "On your <GTM Agent OS> post you said <X> — <question that follows from X>?"
- Not "ask about his hackathon" but "I saw you built a competitive-intel agent at
  Profound's hackathon — how are you feeding it account context across the AI engines?"

## Watch-outs
Sensitivities, unresolved items, anything to avoid. [timeline]

## Next step
The one move this meeting should produce.
```

### 7. Write the brief back to Nous (always — this is what makes it compound)

Save the **full brief** onto the account with the `save_note` MCP tool, so it is
a durable, retrievable document the next run (step 2) and any future agent can
read:

- `focus`: the person's email, LinkedIn URL, or entity UUID (not a bare name).
- `type`: `meeting_brief`.
- `title`: `Pre-meeting brief — <meeting name> <YYYY-MM-DD>`.
- `date`: the meeting date.
- `content`: the **entire brief text** (every section), plus a `Sources:` line.
  Put the whole thing in `content` — that's the part future agents read.

This is non-negotiable: a brief that isn't saved can't be retrieved next time,
and the whole point is that each meeting starts from the last one. Because
step 2 reads these back, re-running for the same person picks up where you left
off instead of starting cold.

(Also fire one `record` event — `interaction.meeting` with kind `event` — so the
meeting itself lands on the timeline, separate from the saved document.)

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
On the account in Nous as a saved `meeting_brief` note (via `save_note`), and in
chat. Because it's saved, the next brief for this person retrieves it and
continues from it. Add a Google Docs connector to also drop it into a doc.
