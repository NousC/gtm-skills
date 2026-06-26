---
name: signal-scan
description: One job — scan an account (or a whole Nous lead list) for buying signals from the company website plus what Nous already knows, rank them, record a structured signal block onto the Nous account (so it powers ICP scoring), AND save a comprehensive signal brief in Notes — each signal carrying the named copy-variables (raise_size, founder_handle, buyer_post_url…) that the campaign-writer later injects. This is the first enrichment pass: signals are the features the ICP model scores on, and the brief is the copy-fuel outreach reads. Use it when the user says "scan my list for signals", "find buying signals on these accounts", "research and prioritize this list", "what's going on at these companies", "enrich my leads with signals", or before scoring/working a list. It does ONE thing and hands off: it does NOT scrape LinkedIn posts (that's the separate post-scraper skill, run later only on ICP-qualified leads) and does NOT write outreach (that's the cold-email / linkedin skills). For a single 1:1 pre-meeting brief use meeting-brief instead.
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
(pain-led / value-led / fallback) · **angle** (one line) · **key data points for
copy** (see below). Rank most exclusive/highest-intent first; pick the **anchor**
(strongest). Always include a **fallback** for when no strong signal lands.

**Key data points for copy — the most important new output.** A signal isn't just
a reason to reach out; it's structured *copy-fuel*. For every signal, extract the
**named variables** the email will inject — the exact strings, not vague prose:

```
raise_size: "$140M Series C"          founder_handle: "Alex Jekowsky, Co-founder & CEO"
raise_date: "March 26, 2026"          founder_ai_thesis_quote: "horizontal AI … hit a wall"
lead_investor: "Sumeru Equity"        buyer_post_url: "https://linkedin.com/posts/…"
expansion_verticals: "dry cleaners…"  hiring_role: "VP Marketing"
```

These are what the email-writer NAMES in the copy. **Every variable AND every
signal's `detected` must be a specific named fact, never an abstraction** — the
email is only as concrete as what you capture. The bar:
- funding → "$140M Series C, March 2026, Sumeru lead", NOT "recently funded"
- stack → "Clay + Smartlead + Apify, Smartlead Certified Partner", NOT "modern tooling"
- proof → "Hook+Ladder: 20+ meetings in 90 days", NOT "has case studies"
- verticals → "dry cleaners, route operators, tailors, cobblers", NOT "several verticals"
- hiring → "3 SDR roles posted in 30 days", NOT "they're growing"

If a field comes out vague, the source was too thin — dig further (another page,
web search, the careers/about page), don't record fluff. NOTE: the email-writer
NAMES these observationally ("saw you closed a $140M Series C"), it never quotes a
person's literal words back at them. Capture the facts so the copy can name them.

### 5. Record the signals onto the COMPANY (not the person)
These six classes are **company facts** — stack, hiring, momentum, friction,
domain are shared by everyone at the company. You'll reach five people at one
agency (Founder, Head of Growth, Marketing VP…); the company signal is the same
for all of them, so it belongs on the **company entity**, not duplicated per
person. The *person's* unique signal (their voice / intent) comes later from
`content-scan` and lands on the person. The person record then **inherits** the
company signals automatically, so each person shows: company signals (inherited) +
their own intent.

Use the **`record_signal`** MCP tool. One call per signal, the **strongest one per
class** (stored as the current `signal.<class>` fact, one per class):
- `focus` — **the company DOMAIN** (e.g. `acme.com`), so it resolves to the company
  entity shared by all its people. Not a person email.
- `signal_class` — stack | hiring | momentum | friction | domain (intent comes from
  `content-scan`, on the person — don't write intent here)
- `detected` — the specific, factual finding
- `implies` — their day-to-day reality (optional)
- `score` — 0-10
- `approach` — pain_led | value_led | fallback (optional)
- `angle` — one-line outreach angle (optional)
- `variables` — the key data points for copy (see step 4), in the brief

This writes a structured `signal.<class>` claim on the **company** that (1) shows
under the company's **Signals tab**, (2) is **inherited by every person** at that
company, and (3) **feeds the ICP scorecard as a feature**. The comprehensive brief
(5b) is also saved on the company; per-person copy fuel (voice/intent) is added by
`content-scan` on each person.

### 5b. Save the comprehensive signal brief as a note (the copy fuel)
The structured signals (5) feed *scoring*; the **brief** feeds the *writer*. After
recording the signals, save ONE comprehensive markdown brief via the **`save_note`**
MCP tool on the same account. This is the single document `campaign-writer` reads to
write the copy, so it must carry the **named copy-variables**, not just prose — and
it must follow this **exact structure** every time, so every brief across every
account is consistent and machine-readable. Title it `Signal brief · <today>`.

```markdown
## Signal Scan: <Company> (<domain>)   ICP fit: <score>/100
**Summary:** <1-2 sentences: what they do, current situation, most interesting finding>

### Signal 1: <name> (Score X/10, <class>)
**Detected:** <specific, factual, dated finding>
**Situation it implies:** <their Monday-morning reality — what the buyer is dealing with>
**Recommended approach:** Pain-led | Value-led | Fallback
**Campaign angle:** <one sentence — the core message this signal enables>
**Key data points for copy:**
- <variable_name>: "<exact value>"
- <variable_name>: "<exact value>"

### Signal 2 … 3 …   (same structure, strongest first)

### Fallback (Score X/10)
**Situation assumption:** <most common pain for this profile when no strong signal lands>
**Campaign angle:** <one sentence>
**Key data points for copy:**
- <variable_name>: "<exact value>"
```

Why comprehensive (not the old 2-3 liner): the brief **is** the instruction-set
`campaign-writer` compiles into the email. The **Key data points for copy** are its
heart — named variables with exact values. The structured `signal.*` records stay
the source of truth for *scoring*; this note is the source of truth for *writing*.
Together: 6 signals (machine-scored) + 1 brief (the writer's input) per account.

### 6. Show the readout (chat only, not saved to a file)
Print the structured scan so the operator sees it now:

```
## Signal Scan: <Company>   ICP fit: <score>/100
Summary: <1-2 sentences — what they do, their situation, the most interesting find>

Signal 1: <name>  (Score X/10, <class>)
  Detected:   <factual finding>
  Implies:    <their Monday-morning reality>
  Approach:   Pain-led | Value-led | Fallback
  Angle:      <one sentence the signal enables>
  Copy vars:  raise_size="$140M" · founder_handle="…" · buyer_post_url="…"

Signal 2 … 3 …

Fallback (Score X/10)
  Assumption: <most common pain for this profile>
  Angle:      <one sentence>
  Copy vars:  <named variable>="<value>"
```

This chat readout mirrors the brief saved in 5b — same fields, so what the operator
sees is exactly what `campaign-writer` will read.

When scanning a list, also print a one-row-per-account table (account · ICP fit ·
anchor signal · score) and note which cleared the ICP threshold — those are the
ones to send to the post-scraper next.

## Running at scale — scout once, fan out, assemble

For a single account, just run steps 1-6 inline. For a **list (~10+ accounts)**,
don't do them one-by-one and don't make every account re-do the shared work. Fan
out instead — accounts are independent and every write goes to Nous per-entity
(`record_signal` / `save_note` keyed to that account), so there's **no shared
local file and no collision** when many run at once.

1. **Scout once (you, the main agent).** Resolve the list and pull the accounts.
   Load the ICP lens a **single time** — step 1 (`get_icp_model` / read
   `context/icp.md`). Do this ONCE, not per account.
2. **Fan out — one sub-agent per account, in capped batches (~8-10 at a time).**
   Give each a tight prompt and **paste the ICP lens in** so it never re-fetches:
   > "Run signal-scan on this ONE account: `<email/domain>`. ICP lens: `<paste>`.
   > Do steps 2-5b: `get_context` → `WebFetch` the site → classify the six signal
   > classes → for each, extract the **key data points for copy** (named variables,
   > exact values) → `record_signal` the strongest per class → `save_note` the
   > **comprehensive brief** (the exact 5b structure, with Key data points for copy).
   > Return ONLY the readout row: account · ICP fit · anchor signal · score."
   Use the `abm-operator` sub-agent (or a general one). Cap concurrency at ~8-10;
   batch the rest.
3. **Assemble (you).** Collect the rows into the one-row-per-account table and
   flag who cleared the ICP threshold (≈70) — that's the hand-off list for
   content-scan.

Why inject the lens: it keeps every account judged against the **same** ICP and
saves N redundant `get_icp_model` calls. For a **handful (≤5)** the fan-out isn't
worth the overhead — run them inline.

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
- **Nous is the one source of truth.** Both outputs live in Nous — the structured
  signals (`record_signal`) and the comprehensive brief (`save_note`). Never a
  separate local file. Signals are the source of truth for *scoring*; the brief is
  the source of truth for *writing*.
- **Website-only research here.** For a 1:1 pre-meeting brief that reads their
  posts, use meeting-brief.

## What runs after this (not here)
1. **ICP scoring** — Nous scores fit from the `signal.*` features you recorded.
2. **Post-scraper skill** — on ICP-qualified leads (>70) only, adds deep Intent
   from LinkedIn posts.
3. **cold-email / linkedin** — write the message from the anchor signal.
