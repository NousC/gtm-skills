---
name: content-scan
description: The deep Intent layer. Scrape a prospect's recent LinkedIn posts via Apify, read them semantically for the themes that map to your offer, then record a distilled Intent signal on the Nous account AND save the quoted evidence as a note for personalization. This is the second enrichment pass, run AFTER signal-scan + ICP scoring, and ONLY on ICP-qualified leads (≈70+) because it costs money per profile. Use it when the user says "scrape their posts", "what's [prospect] been posting about", "find intent in their LinkedIn content", "deep-enrich my qualified leads", or "has this list been talking about [theme]". It does ONE thing: posts → Intent. It does not scan websites (that's signal-scan) and does not write outreach (that's cold-email / linkedin). For a single 1:1 pre-meeting brief use meeting-brief instead.
---

# Content scan

## One job

Scrape a prospect's recent **LinkedIn posts** and turn them into **Intent** — the
strongest evidence that they're consciously working the problem your offer solves.
Two outputs, both on the Nous account:

1. **A distilled Intent signal** → `record_signal` (`signal.intent`, scored, with
   an angle drawn from a real post). The feature that feeds the ICP model + shows
   on the Signals tab.
2. **The evidence as a note** → `note.post_scan` (the quotes + post links +
   themes), like `meeting-brief` saves a brief. Rich for personalization; the
   `cold-email` / `linkedin` skills read it. Shows under the Notes tab.

No local file — Nous is the one source of truth.

This runs **after** signal-scan + ICP scoring, **only on ICP-qualified leads
(≈70+)**, because it's paid (Apify, ~$0.005/post). It's the deep layer; signal-scan
is the free first pass.

The boundary, on purpose:
- **This skill:** LinkedIn posts → Intent signal + evidence note.
- **signal-scan (before this):** website + Nous data → the six signal classes, free.
- **cold-email / linkedin (after this):** write the message. Not here.

## How to invoke

`/content-scan` — or *"scrape posts for my ICP-qualified leads"*, *"what's Georgi
been posting about"*, *"has this list talked about cold email being broken?"*.

Resolve from the prompt:
- **Target** — one prospect, or a set of ICP-qualified leads. **Resolve by
  LinkedIn URL or email, never the display name** (name lookup 404s).
- **Theme** *(optional)* — a specific topic to hunt for. If omitted, scan for the
  themes that map to your offer (from the GTM profile).
- **Count** — posts per profile (default 20).

## First-run setup

- **Nous** — try `get_context`. Not connected → set up the MCP (see signal-scan).
- **Apify** — needs `APIFY_API_TOKEN` (or `APIFY_TOKEN`). Missing → "I read posts
  through Apify, ~$0.005/post (~$0.10 per profile for 20). Add the token:
  `export APIFY_API_TOKEN=apify_api_xxx` (apify.com → Settings → Integrations)."

Actor: `apimaestro/linkedin-profile-posts` (URL form `apimaestro~linkedin-profile-posts`) — no LinkedIn cookies needed.

## Core philosophy

**Intent is what they say in their own words.** Quote real posts; never
paraphrase into a fake signal. If they haven't posted on-theme, say so — silence
is information.

**Match semantically, not by keyword.** "We're ripping out our playbooks for
Claude agents" matches "AI-first GTM" without the phrase. A passing "AI" mention
in an unrelated post does not.

**Judge against your offer.** A post is Intent only if it touches the problem you
solve (load the GTM profile first). Otherwise it's noise.

**Spend on keepers.** Paid, so ICP-qualified only, with a cost preview.

## The process

### 1. Load your offer/ICP lens — from the user's own ICP file
The ICP now lives in the user's workspace, not a web form. **Read their ICP file**
(`context/icp.md` first; else `.claude/`, `gtm/`, `icp*`/`positioning*`) — the prose
defines the problem you solve, and the `<!-- nous:icp -->` block holds the learned
model of which signals predict a win. Call `get_icp_model` to refresh that block
(`get_icp` re-syncs if the file changed); use `get_gtm_profile` only to fill gaps
not in the file. The problem you solve defines which themes count as Intent.

### 2. Resolve the prospect + gate on ICP
`get_context` by **email or LinkedIn URL** (never name) → confirm ICP fit ≥ ~70
and grab the `linkedin_url`. Skip leads below the threshold — they don't earn a
paid scan. For a list, show the cost preview before spending:
> "N ICP-qualified leads · scan posts ≈ $X (N × 20 × $0.005). Run it?"

### 3. Scrape recent posts (paid)
One Apify run per profile, in parallel:

```bash
curl -s -X POST \
  "https://api.apify.com/v2/acts/apimaestro~linkedin-profile-posts/run-sync-get-dataset-items?token=$APIFY_API_TOKEN&timeout=120" \
  -H "Content-Type: application/json" \
  -d '{ "username": "<linkedin-username-or-url>", "total_posts": 20 }'
```
`username` works with a bare handle (`georgi-furnadzhiev`) or a full profile URL
(confirmed). If a run returns 400/empty, try `profileUrl` / `profileUrls` instead;
adapt and retry once. Don't loop paid runs. Reshared posts count — read them too.

### 4. Read for Intent (semantic)

> **Capture SPECIFIC NAMED FACTS, not abstractions. This is the whole point.** The
> email-writer can only NAME what you capture. If you write "they post about
> outbound", the email comes out abstract and gets deleted. Pull the concrete,
> nameable things: **exact numbers** ("170k emails/mo", "20+ meetings in 90 days",
> "36% positive reply rate"), **named tools** (Clay, Smartlead, Apollo, Claygent,
> not "their stack"), **named clients / companies / case studies**, **named
> verticals / ICPs**, and the **specific public stances** they've taken (the actual
> position, e.g. "argues open rates are vanity, only positive replies count" — NOT
> the vague "cares about quality"). A fact you can drop into a sentence verbatim is
> usable. A theme is not. Zevenue's bar: "Google Maps + Mindbody + state license
> registries", never "the data sources".

Two reads:
- **What they post about overall** — their dominant themes + voice, AND the specific
  named stances/claims they repeat (named, observational, never to be quoted back).
- **On-theme matches** — for each post that touches the problem your offer solves,
  capture: **date**, the **specific named facts in it** (numbers, tools, claims), the
  **post URL**, and a **one-line why-it-matches**. The quote is for YOUR understanding;
  what the writer uses is the named facts, never the quote cited back at them.
Pick the **anchor** — the single strongest, most recent on-theme post.

### 5. Record — a SHORT signal + a LONGER structured note (both on Nous, no file)
Different jobs, different lengths.

- **The Intent signal** → `record_signal` (**keep it short** — it's a glanceable
  line on the Signals tab): `signal_class: "intent"`, `detected` = ONE tight
  sentence naming the theme (a 3-6 word quote fragment is fine, not a paragraph),
  `implies` = one short clause, `score` (0-10 by how clearly + recently they work
  the problem), `approach`, `angle` = one line in their words. The detail lives in
  the note, not here. Supersedes any shallow website-Intent.

- **The evidence note** → `save_note` (**the real research** — this is what
  `cold-email` reads): `focus` (email/URL), `type: "research"`,
  `title: "{name} - LinkedIn Post Scan"` (the person's full name + a plain hyphen,
  e.g. "Jordan Crawford - LinkedIn Post Scan"). Put the structured report in `content`:

  ```
  # {name} - LinkedIn Post Scan

  **Scanned:** {date}
  **Theme:** {theme — the problem your offer solves, in your words}
  **Profile:** {linkedin_url}
  **Headline:** {their LinkedIn headline if available}
  **Posts reviewed:** {n}
  **Direct matches:** {m}

  ## What they post about (voice + themes)
  {2-4 sentences: their dominant themes, the *cadence* of what they publish, and
  their VOICE — how they write (contrarian? data-led? teacher? story?). This is the
  copy fuel: the email-writer mirrors this voice. Be specific, quote a phrase or two
  they actually use.}

  ## Direct matches
  ### {date} — [post]({post_url})
  > {1-3 sentence quote in their own words — the real line, not a paraphrase}

  **Why it matches:** {one line — the specific link to the problem you solve}
  **Copy hook:** {one line — the exact angle/phrase the email can echo back to them}
  {repeat per on-theme post, strongest + most recent first}

  ## Adjacent signals
  {posts that touch the space without being dead-on — same shape + a "Why it's
  adjacent" line. Only if any.}

  ## Voice & phrasing bank
  {3-6 verbatim short phrases / words they actually use — the vocabulary the email
  should sound like. e.g. "leverage", "the math nobody does", "is this you". This is
  what makes the copy sound like a peer, not a vendor.}

  ## Anchor
  {the single strongest, most recent on-theme post — date + one-line why it's the
  lead angle for outreach.}

  ## No-match summary
  {only if zero on-theme posts — one sentence on what they DO post about, so you can
  judge whether the theme is truly absent or just framed differently.}
  ```
  (save_note takes focus/title/content/type/date only — no category/metadata.)
  **Make it comprehensive** — this note is the entire input the email-writer compiles
  into copy. More detail = better, more personal emails. Never thin it out.

### 6. Report back (chat only)
Per profile: `{name}: {m} on-theme posts — anchor: "{short quote}"` or
`{name}: no on-theme posts (they mostly post about {X})`. Note the strongest find.
Don't paste the full evidence — it's on the Nous record.

## Running at scale — gate once, scout once, fan out, assemble

Same scout → fan out → assemble shape as signal-scan, with **two paid gates up
front** because this skill spends money. For a single prospect, run steps 1-6
inline. For a **set of qualified leads**, fan out — profiles are independent and
all writes go to Nous per-entity, so no shared file, no collision.

1. **Gate + scout once (you, the main agent).** Filter to **ICP ≥70 only** (step
   2). Load the ICP lens a **single time** (step 1). Show **one cost preview and
   get one yes** for the whole batch — never let sub-agents ask individually:
   > "N qualified leads × 20 posts ≈ $X (N × 20 × $0.005). Run it?"
2. **Fan out — one sub-agent per qualified profile, capped at ~8-10.** Paste the
   ICP lens in so none re-fetch:
   > "Run content-scan on this ONE profile: `<linkedin-url/email>`. ICP lens:
   > `<paste>`. Do steps 3-5: Apify scrape → read posts semantically for on-theme
   > Intent → `record_signal` (`signal.intent`) + `save_note` the evidence.
   > Return ONLY: `{name}: {m} on-theme — anchor "{quote}"` (or no on-theme posts)."
   Use the `abm-operator` sub-agent (or a general one); batch the rest.
3. **Assemble (you).** Collect the one-liners; call out the strongest finds.

Confirmation happens **once, at the top** — sub-agents only ever run on
already-approved, already-qualified leads. For a **handful (≤5)** run inline.

## Hard rules
- **Quote, never invent.** No Intent without a real post behind it. No on-theme
  posts → say so plainly (the silence is the finding).
- **Semantic, not keyword.** Match meaning; ignore passing mentions.
- **ICP-qualified only, confirm before spending.** Never run the paid scrape on
  sub-threshold leads or without a cost preview + yes.
- **One job.** Posts → Intent. No website scan (signal-scan), no message writing
  (cold-email / linkedin). Profiles only — not company pages.
- **Resolve by URL/email, never name.**
- **Nous is the one source of truth.** Signal + note on the account; no local file.

## What runs after this
**cold-email / linkedin** — write the message from the anchor signal + the
`note.post_scan` quotes (real words → a message that sounds like a peer).

## Cost
~$0.005/post → ~$0.10 per profile (20 posts), ICP-qualified leads only, after
confirmation. 50 qualified leads ≈ $5.
