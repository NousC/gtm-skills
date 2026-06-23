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
Two reads:
- **What they post about overall** — 1–2 sentences on their dominant themes +
  voice (so we know what they write about, even when nothing is on-theme).
- **On-theme matches** — for each post that touches the problem your offer solves,
  capture: **date**, a **1–3 sentence quote in their own words**, the **post URL**,
  and a **one-line why-it-matches**. Borderline → an "adjacent" note.
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
  `title: "Post scan — {date}"`. Put the structured report in `content`:

  ```
  # Post scan — {themes} · {date}
  {profile_url} · {n} posts reviewed · {m} on-theme

  ## What they post about
  {1-2 sentences: their dominant themes + voice — so we know what they write
  about even when nothing is on-theme}

  ## Direct matches
  ### {date} — {post_url}
  > {quote in their own words}
  Why it matches: {one line}
  {repeat per match}

  ## Adjacent signals
  {only if any — same shape + a "Why it's adjacent" line}

  ## No on-theme posts
  {only if zero matches — one sentence on what they DO post about, so you can
  judge whether the theme is truly absent or just framed differently}
  ```
  (save_note takes focus/title/content/type/date only — no category/metadata.)

### 6. Report back (chat only)
Per profile: `{name}: {m} on-theme posts — anchor: "{short quote}"` or
`{name}: no on-theme posts (they mostly post about {X})`. Note the strongest find.
Don't paste the full evidence — it's on the Nous record.

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
