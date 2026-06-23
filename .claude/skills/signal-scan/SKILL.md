---
name: signal-scan
description: One job — scan an account (or a whole Nous lead list) for buying signals from the company website plus what Nous already knows, rank them, and record a concise signal block straight onto the Nous account so it powers ICP scoring. This is the first enrichment pass: signals are the features the ICP model scores on. Use it when the user says "scan my list for signals", "find buying signals on these accounts", "research and prioritize this list", "what's going on at these companies", "enrich my leads with signals", or before scoring/working a list. It does ONE thing and hands off: it does NOT scrape LinkedIn posts (that's the separate post-scraper skill, run later only on ICP-qualified leads) and does NOT write outreach (that's the cold-email / linkedin skills). For a single 1:1 pre-meeting brief use meeting-brief instead.
---

# Signal scan

## One job

Scan an account (or every account in a Nous lead list) for **buying signals** —
concrete, current facts about what situation the company is in — rank them, and
**record a concise signal block onto the Nous account.** That's the whole job.

Why it exists: signals are the **features the ICP model scores on**. You can't
score fit well without them. signal-scan is the first enrichment pass that gives
Nous something real to score. Everything downstream (the deeper post-based Intent
layer, the outreach copy) is a **different skill** that runs after this.

The boundary, on purpose:
- **This skill:** signals from website + Nous data → recorded on the account.
- **Post-scraper skill (separate, later):** LinkedIn posts → deeper Intent, run
  ONLY on leads that score above the ICP threshold (≈70). Not here.
- **cold-email / linkedin skills:** write the message. Not here.

One source of truth: **Nous.** The concise signal block is written onto the
account — no separate local file. The full readout below is shown in chat at
runtime; the record that persists is the concise one on the Nous account.

## How to invoke

`/signal-scan` — or *"scan my 'GTM founders' list for signals"*, *"what's going on
at the accounts in my LinkedIn Engagers list"*.

Resolve from the prompt:
- **Target** — one account/domain, or a Nous lead-list name/id (ask + list the
  lists if unclear).
- **Scope** — whole list, or a cap (e.g. top 100). Default: whole list.

## First-run setup (you, the agent, run once)

Try the Nous `get_context` tool.
- Connected → go.
- Not → set it up: `claude mcp add nous -e NOUS_API_KEY=<pk_…> -- npx -y @opennous/mcp`
  (key at opennous.cloud → Settings → API keys). Confirm `get_context` works.

No Apify, no other paid API. Website signals use plain `WebFetch` — free.

## The six signal classes

Every signal belongs to one class. Score each **1-10** (heuristic prior; the
weekly audit later replaces it with the learned weight).

| Class | What it captures | Where (website + Nous) |
|---|---|---|
| **Stack** | tools they run, competitor usage, duct-taped processes | integrations, footer, pricing, Nous facts |
| **Hiring** | roles posted, esp. ones signalling strain or that your buyer owns | /careers, /jobs |
| **Momentum** | funding, expansion, new markets, headcount growth | homepage news, /about, Nous facts |
| **Friction** | public complaints, broken-process tells | reviews, support pages, blog |
| **Intent** | site/blog language showing they're working the problem | /blog, news *(deep post-based Intent = the post-scraper skill, later)* |
| **Domain** | vertical/marketplace tells unique to their niche | homepage, Nous facts |

Strength guide: 8-10 behavioural + exclusive (direct competitor with friction;
hiring the role you replace) · 5-7 adjacent tool / growth pattern · 3-4
size/stage suggests it · 1-2 basic ICP fit only.

## The process

Per account, in order. Note thin sources — never fabricate a signal.

### 1. Load your offer/ICP lens — from the user's own ICP file
**The ICP now lives in the user's workspace, not a separate web form.** Nous and
the user's repo are in symbiosis: their ICP prose lives in their own file
(`context/icp.md`, or wherever their GTM context sits — `.claude/`, `gtm/`), and
Nous writes the **learned scoring model** back into that same file inside a
`<!-- nous:icp start -->…<!-- nous:icp end -->` block — the signals that predict a
win, with their weight and lift. **That block is the lens for this skill.**

Do this, in order:
1. **Read the user's ICP file** with your file tools (`context/icp.md` first;
   else look in `.claude/`, `gtm/`, or files named `icp*` / `positioning*`). The
   prose gives you the offer + buyer; the `nous:icp` block tells you **which of
   the six signal classes actually predict wins for this ICP, and how heavily** —
   weight the scan toward those classes.
2. Call **`get_icp_model`** to get/refresh that learned block; if the file changed
   since Nous last saw it, **`get_icp`** re-syncs it in.
3. For product / competitors / positioning that aren't in the ICP file,
   `get_gtm_profile` still fills the gaps.

This decides WHICH signals matter: a hiring signal is noise unless the learned
model (or the offer) says that's a problem you solve. Judge every signal against
*this* ICP. If there's no ICP file yet, fall back to
`~/Desktop/AIOS/AIS-OS/context/positioning.md` and offer to create
`context/icp.md` from what the user tells you, so the ICP lives in their repo
going forward.

### 2. Pull what Nous already knows
`get_context` (intent `account_review`) → known facts, ICP fit (0-100 + why),
company. Don't re-derive what Nous already has; build on it.

> **Resolve each lead by its email or LinkedIn URL, never the display name.**
> Name lookup is unreliable and 404s (`entity_not_found`). Lead lists carry the
> email / `linkedin_url` — use those as `focus` (or the entity UUID). Same for
> `record` / `record_signal`.

### 3. Scan the website (free)
Resolve the domain. Use the built-in **`WebFetch`** tool on the homepage plus
`/about`, `/careers` (or `/jobs`), `/product`, `/pricing`, `/blog`,
footer/integrations where they exist. Read them for the six classes.

> `WebFetch` is Claude Code's built-in fetcher (free, no API key) — **not
> Firecrawl.** If a JS-heavy site returns little, say the source is thin. Optional
> upgrade: if `FIRECRAWL_API_KEY` is set, use Firecrawl's scrape/crawl for
> JS-rendered sites instead — richer, but paid. Default to free `WebFetch`.

### 4. Identify + rank signals
Read **`references/signal-types.md`** — the full catalog (the six classes, where
to find each, the 1-10 scoring rubric, and the patterns that amplify). For each
signal capture: **detected** (specific, factual) · **implies** (their day-to-day
reality) · **score** (1-10, per the rubric) · **class** · **approach**
(pain-led / value-led / fallback) · **angle** (one line). Rank most
exclusive/highest-intent first; pick the **anchor** (strongest). Always include a
**fallback** for when no strong signal lands.

### 5. Record the signals onto the Nous account
Use the **`record_signal`** MCP tool — the canonical, validated way to write a
signal. One call per signal, the **strongest one per class** (it's stored as the
current `signal.<class>` fact, so there's one per class):
- `focus` — the person's email or entity id
- `signal_class` — stack | hiring | momentum | friction | intent | domain
- `detected` — the specific, factual finding
- `implies` — their day-to-day reality (optional)
- `score` — 0-10
- `approach` — pain_led | value_led | fallback (optional)
- `angle` — one-line outreach angle (optional)

This writes a structured `signal.<class>` claim that (1) shows under the **Signals
tab** on the account and (2) **feeds the ICP scorecard as a feature** (signal
claims flow into the feature map the scorer reads). One source of truth, one
write — no notes, no local file.

### 6. Show the readout (chat only, not saved to a file)
Print the structured scan so the operator sees it now:

```
## Signal Scan: <Company>   ICP fit: <score>/100
Summary: <1-2 sentences — what they do, their situation, the most interesting find>

Signal 1: <name>  (Score X/10, <class>)
  Detected:  <factual finding>
  Implies:   <their Monday-morning reality>
  Angle:     <one line the signal enables>

Signal 2 … 3 …

Fallback (Score X/10)
  Assumption: <most common pain for this profile>
  Angle:      <one line>
```

When scanning a list, also print a one-row-per-account table (account · ICP fit ·
anchor signal · score) and note which cleared the ICP threshold — those are the
ones to send to the post-scraper next.

## Hard rules

- **Be specific, not generic.** "They're growing" is not a signal. "Posted 3
  supply-chain coordinator roles in the last 30 days" is. A signal is a concrete,
  dated, factual finding.
- **Connect every signal to the offer.** It only matters if it indicates a
  problem your offer solves (step 1 is the lens). Otherwise it's noise.
- **Situations over demographics.** Describe what their team is dealing with
  day-to-day (the `implies` field), not what the company looks like on paper.
- **Score honestly.** A 4/10 is useful — it tells the downstream writer to go
  softer. Don't inflate.
- **Always include a fallback.** Even with strong signals, add the
  most-common-pain fallback for the rest of the segment.
- **Flag what you couldn't find.** Pages unavailable, JS-heavy site, thin data →
  say so plainly. Never fabricate to fill a gap.
- **Enrichment + Nous facts trump website guesses.** If Nous already knows
  something (`get_context`) or the user passes Clay/Apollo/BuiltWith data,
  prioritize that over inferences from the site.
- **One job.** Signals only — no LinkedIn-post scraping (separate skill), no email
  writing (separate skill). If asked, point at the right skill and stop.
- **Nous is the one source of truth.** Record concise signals onto the account;
  do not write a separate report file.
- **Website-only research here.** For a 1:1 pre-meeting brief that reads their
  posts, use meeting-brief.

## What runs after this (not here)
1. **ICP scoring** — Nous scores fit from the `signal.*` features you recorded.
2. **Post-scraper skill** — on ICP-qualified leads (>70) only, adds deep Intent
   from LinkedIn posts.
3. **cold-email / linkedin** — write the message from the anchor signal.
