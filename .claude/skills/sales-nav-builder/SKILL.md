---
name: sales-nav-builder
description: Turn a LinkedIn Sales Navigator search into a complete, ICP-scored lead list in Nous, paying for emails only on the leads worth contacting. Claude builds the search structural-filters-first (years at company, headcount, the right industries) with modern-vocabulary keywords second, Evaboot extracts the whole search with NO emails, and then EVERY extracted lead is imported into Nous and ICP-scored in place — nothing is ever dropped, because you paid to extract it. The ICP score is the filter, living inside the list. Only the ICP-qualified leads then get verified emails found via Prospeo straight from the LinkedIn URL (Evaboot as fallback), so the variable cost lands on keepers. For people who have Sales Navigator.
---

# Sales Nav lead builder

## What it does

You have **LinkedIn Sales Navigator** — the sharpest targeting filters that
exist. This skill turns a Sales Nav search into a **complete, ICP-scored** lead
list in Nous, and spends email credits **only on the leads worth contacting.**

The core principle, learned the hard way on a real 1,205-lead run:

> **You paid to extract every lead, so import every lead.** Never pre-filter
> leads out of the list. Score all of them, label all of them, and let the
> **ICP score be the filter inside Nous.** Deletion is the operator's manual
> choice, never the skill's.

The money model: **extraction** is the paid step you can't avoid (1 credit per
lead). **Scoring, importing, and labelling are free** in Nous. **Email-finding**
is the second paid step, and it runs only on the ICP-qualified keepers.

```
Stage 1  EXTRACT (pay)   Evaboot, enrich_email="none" → LinkedIn data, NO emails.
                         1 credit / lead, deducted on completion.
Stage 2  SCORE (free)    Pull the Nous GTM profile → ICP-score EVERY lead 0–100.
                         Evaboot's "Matches Filters" + the blocklist are SIGNALS,
                         not gates. Dedup marks net_new — it does not drop.
Stage 3  IMPORT (free)   Save ALL leads into one Nous list, each labelled
                         icp / icp_score / icp_reason. Nothing discarded.
Stage 4  EMAILS (pay)    Prospeo (off the LinkedIn URL) finds the verified email,
                         ONLY on ICP-qualified net-new leads. 1 credit / email
                         found (misses free). Evaboot Email Finder is the fallback.
```

The flow: **Claude (structural search) → Sales Nav → Evaboot extract (no emails)
→ pull leads via API → Nous (score EVERYTHING) → import all → emails on keepers.**

## How to invoke

`/sales-nav-builder` — or *"build me a list of outbound GTM agency founders from
Sales Navigator."*

## First-run setup (run once as a short interview)

**1. LinkedIn Sales Navigator (required).** Confirm a seat. Evaboot scrapes
**through your own connected Sales Nav session** — that is what keeps it
account-safe and GDPR-clean. No Sales Nav → point them to `lead-builder`.

**2. Evaboot (required) — the extractor.** Check `EVABOOT_API_KEY`
(dashboard → Settings → API). Then **verify the account is ready** with the free
quota call — `has_valid_salesnav` MUST be `true`, or extraction will fail no
matter how many credits exist:

```bash
curl -s "https://api.evaboot.com/v1/quota/" -H "Authorization: Bearer $EVABOOT_API_KEY"
# → { "quota": { "has_valid_salesnav": true, "credits": 1500.0, ... } }
```

If `has_valid_salesnav` is `false` / `salesnavs: []`, the user must **connect
their Sales Nav once**: install the Evaboot Chrome extension and sign in / run
one export with the LinkedIn account that holds the Sales Nav seat. After that,
the headless API works. This is a one-time connect, not a per-run step.

**2b. Prospeo (required) — the email finder.** Check `PROSPEO_API_KEY` (used with
an `X-KEY` header). Prospeo finds the verified email **directly from the LinkedIn
URL**, so it needs no domain and covers leads Evaboot can't (no-domain ones). It
is also cheaper per email (~$0.02 at volume vs ~$0.04–0.05 for Evaboot) and
charges **only on a found email** (misses/errors are free — verified live). Quick
balance check:

```bash
curl -s -X POST "https://api.prospeo.io/account-information" \
  -H "X-KEY: $PROSPEO_API_KEY" -H "Content-Type: application/json" -d '{}'
# → { "response": { "remaining_credits": N, ... } }
```

Evaboot's Email Finder stays available as a fallback for misses.

**3. Nous — the ICP, dedup, and where the list lands (required).** Check
`NOUS_API_KEY` (`pk_`) and connect the MCP for `get_gtm_profile`. Nous holds the
**ICP / GTM context** the skill scores against, dedups, and stores the list. The
scoring runs in the skill (your Claude tokens), not on Nous.

---

## Phase 1 — Build the search: structural filters first, keywords second

The biggest mistake is leading with keywords. Legacy local agencies and modern
AI-native GTM agencies **both** say "lead generation" — keywords can't separate
them. **Structure can.** Build the search in this order:

**1. Years in current company ≤ 5 — the single biggest lever.** Removes the
16-to-24-year-tenure legacy marketing, SEO, and design veterans; keeps founders
who started modern agencies.

**2. Company headcount** — the size band from the saved ICP (e.g. 1–10).

**3. The right industries — and the trap to avoid.** For modern AI-native GTM
agencies, do **not** use **"Advertising Services"** — that is the legacy
word-of-mouth / SEO / web-design / branding trap. The real ICP founders are
tagged under:
> **Marketing Services · Business Consulting & Services · IT Services & IT
> Consulting · Software Development · Technology, Information & Internet**

**4. Seniority / title** — Founder, Co-founder, CEO, Owner.

**5. Keywords — a FLAT POSITIVE `OR` list, modern vocabulary only:**
> `GTM OR "go-to-market" OR Clay OR Claude OR "AI-native" OR "AI SDR" OR RevOps
> OR outbound OR "cold email"`

### Hard keyword rule — never break this

Sales Nav's keyword box **breaks on Boolean `NOT`.** A long `(...) AND NOT (...)`
returns **0 results** every time. **Never put exclusions in the keyword box.**
Flat positive `OR` only; all judgment happens downstream in Stage 2.

### Optional — let Evaboot build the search URL for you

`POST /v1/search-builder/` turns a natural-language description into a Sales Nav
search URL with structural filters:

```bash
curl -s -X POST "https://api.evaboot.com/v1/search-builder/" \
  -H "Authorization: Bearer $EVABOOT_API_KEY" -H "Content-Type: application/json" \
  -d '{ "description":"AI-native GTM / outbound agency founders, 1-10 employees, North America", "search_type":"LEAD" }'
# → { "success":true, "url":"https://www.linkedin.com/sales/search/people?...", "filters_used":{...} }
```

Always have the user paste the URL into Sales Nav to **eyeball the result count**
before extracting.

### The 2,500 export cap

Sales Nav silently truncates any export at **2,500 leads**. If the preview count
is above 2,500, tell the user to tighten below it (years band, headcount,
geography) so the pull is complete.

## Phase 2 — Extract LEADS ONLY (no emails), then pull via API

**Default to leads-only — `enrich_email:"none"`.** Emails are Stage 4, on keepers
only. There are two ways to run Stage 1:

**A. Headless API (preferred).** Trigger the extraction with the Sales Nav search
URL — note the URL must be a `linkedin.com/sales/search/people?...` URL, not a
regular search or profile URL:

```bash
curl -s -X POST "https://api.evaboot.com/v1/extractions/url/" \
  -H "Authorization: Bearer $EVABOOT_API_KEY" -H "Content-Type: application/json" \
  -d '{ "linkedin_url":"https://www.linkedin.com/sales/search/people?query=...",
        "search_name":"gtm-agency-founders", "enrich_email":"none" }'
# → { "search_id":"...", "status":"...", "estimated_leads":N }
```

**B. Chrome extension.** The user runs the search in Sales Nav and clicks
**Export with Evaboot** with email-finding **OFF**. This also registers/links
their Sales Nav session for path A next time.

**Either way, poll until done and pull the leads via the API** (no CSV handoff):

```bash
# list extractions to find the job + watch status (EXECUTING → EXECUTED)
curl -s "https://api.evaboot.com/v1/extractions/" -H "Authorization: Bearer $EVABOOT_API_KEY"
# then pull the full result set
curl -s "https://api.evaboot.com/v1/extractions/<search_id>/" -H "Authorization: Bearer $EVABOOT_API_KEY"
```

Each lead carries rich firmographics: `First Name`, `Last Name`, `Current Job`,
`Company Name`, `Company Domain`, `Company Industry`, `Company Description`,
`Company Employee Range`, `Company Specialities`, `Years in Company`,
`Linkedin URL Public`, `Email` (empty) — and Evaboot's own **`Matches Filters`**
(`YES`/`NO`) + **`No Match Reasons`**.

**Cost reality:** extraction costs **1 credit per lead, deducted on completion**
(not mid-run, and not free). On a 1,205-lead pull that is ~1,205 credits. Check
`/v1/quota/` first and tell the user the extraction cost up front.

## Phase 3 — Score EVERY lead against the ICP (free, no credits)

This stage runs **inside the skill** — you (Claude) read the workspace ICP and
score every lead. **No Evaboot or Nous credits.** The rule that matters:

> **Score everything. Drop nothing.** You paid to extract these leads; they all
> go into the list. The ICP score is the filter, and it lives on the record.

### 1. Pull the full ICP from the Nous GTM context (source of truth)

Prefer the `get_gtm_profile` MCP tool; else:

```bash
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

Use the *whole* profile: who they target, firmographics, signals/vocabulary,
disqualifiers.

### 2. Treat Evaboot's "Matches Filters" and the blocklist as SIGNALS, not gates

Hard-won lesson: **`Matches Filters: NO` does NOT mean off-ICP.** It usually
means the lead didn't literally contain the keyword string you typed — real
RevOps / outbound / GTM founders hide in the `NO` pile. So:

- Keep `Matches Filters` and `No Match Reasons` as **fields on the record**
  (useful signal), but **never filter the list by them.**
- The exclusion list (design, web design, SEO, branding, recruiting, staffing,
  talent, headhunting, PR, social media management, etc.) is a **scoring
  signal**, matched on **company identity (name + industry) with word
  boundaries** — NOT a substring scan of the whole bio. A naive `in` check turns
  "build**ui**ng" into a UI-design hit and "**design**ed" into a design hit, and
  nukes most of your real leads. Use `\b<term>\b`. And even a real hit only
  **lowers the score** — it does not remove the lead.

### 3. Dedup — marks, does not drop

Dedup against Nous before you spend a cent on emails. Evaboot gives you a
`linkedin_url` per lead, so check per-person — that's what splits "buy" from
"re-enrich" from "reuse":

```bash
curl -s -X POST "https://api.opennous.cloud/v2/dedup" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "linkedin_urls": ["https://www.linkedin.com/in/jane-doe", "..."] }'
```

Keep `status` AND the coverage fields (`stale`, `email_status`, `enriched_at`) on
each record — they gate Stage 5 spend, they do NOT drop anyone. Everyone still
enters the list and gets ICP-scored.
- `net_new` → find the email in Stage 5 (if ICP-qualified).
- `needs_enrichment` (you own them, `stale:true` — not enriched in 90 days) →
  Stage 5 Prospeo call still runs; it *re-verifies* the stale email rather than
  buying a brand-new one.
- `reusable` (`stale:false` + `email_status` set) → you already have a fresh
  verified email; **skip the Prospeo call entirely** in Stage 5.

### 4. ICP-score every lead 0–100 against the GTM profile

For **every** lead write into `fields`:
- `icp`: `true | false` — `icp_score >= threshold` (default 40). Kept in fields
  for the list's ICP filter; it is NOT a visible column.
- `icp_score`: `0–100`. This is the single visible **ICP** column.
- `icp_reason`: one short sentence citing what matched or missed. Stored in
  fields for reference; NOT a visible column.

A sensible rubric (combine with judgment): base ~35; **+** for GTM / go-to-market
/ RevOps / outbound / cold-email / SDR / AI-native signals in headline + company
description + title; **+** for founder/owner title and small headcount (≤10);
**−** for off-ICP industries (healthcare, construction, CPG, real estate, PE/M&A,
education, recruiting, design) and large headcount. Score against the **positive
ICP**, not just the blocklist — a company nobody listed still scores low when it
plainly doesn't fit.

## Phase 4 — Import ALL leads into one Nous list, labelled

Create the list, declare the columns, then insert **every** lead in batches
(~400/insert). Emails are empty here — they come in Stage 4 for keepers only.

```bash
# 1. create the list — response carries id AND workspace_id:
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name":"Sales Nav · GTM founders · 1-10 · NA", "source":"sales_nav" }'
# → { "lead_list": { "id":"<LIST_ID>", "workspace_id":"<WS>", ... } }

# 2. declare columns (PATCH REQUIRES workspaceId in body):
# ONE ICP column only — the SCORE, labelled "ICP". Do NOT declare separate
# "icp" (true/false) or "icp_reason" ("Why") columns. `icp` + `icp_reason` still
# go in each lead's `fields` below (icp drives the list's ICP filter); they just
# aren't shown as columns. The score is what you read; the 40+ threshold filters.
curl -s -X PATCH "https://api.opennous.cloud/api/lead-lists/<LIST_ID>" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "workspaceId":"<WS>", "columns":[
        {"key":"icp_score","label":"ICP"}, {"key":"title","label":"Title"},
        {"key":"industry","label":"Industry"}, {"key":"company_size","label":"Size"},
        {"key":"evaboot_match","label":"Evaboot match"} ] }'

# 3. insert EVERY lead, batched ~400 at a time:
curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "workspaceId":"<WS>", "importDuplicates": true, "leads": [
        { "name":"Jane Doe", "linkedin_url":"https://www.linkedin.com/in/janedoe",
          "company":"Acme",
          "fields": { "title":"Founder", "domain":"acme.com", "icp": true, "icp_score": 86,
                      "icp_reason":"AI-native GTM agency, 1-10, NA — core ICP",
                      "industry":"Software Development", "company_size":"2 to 10",
                      "evaboot_match":"NO", "source":"sales_nav" } } ] }'
```

> **Always map Evaboot's `Company Domain` into `fields.domain`** (and `Company
> Employee Range` into `fields.company_size`). The list's Domain column reads
> `fields.domain`, so every lead shows its domain straight from the extract — no
> enrichment needed. Enrichment later just fills the gaps (no-domain leads).

Notes from the live run:
- **Use `importDuplicates: true`** when you intend a complete list — otherwise
  leads that are already workspace contacts get silently skipped, and you lose
  rows you meant to keep.
- **Resolution is asynchronous and scales with size.** ~90 leads settle in
  ~30–60s; ~1,200 leads take several minutes. Reading back immediately shows
  empty `name`/`fields` and `status:"pending"` — that is mid-resolution, NOT a
  failure. Tell the operator the list fills in over a few minutes.
- **The read endpoint pages at ~1,000 leads.** To verify a larger list, page
  with an offset; don't report "missing leads" from a single page.

## Phase 5 — Find emails on the ICP keepers (Prospeo), then write back

Only now, and only on **ICP-qualified keepers that still need an email**:
`icp:true` AND status is `net_new` **or** `needs_enrichment` (the latter just
re-verifies a stale email you already own). **Skip `reusable` keepers** — they
already have a fresh verified email — and skip non-ICP leads. That's how the
Prospeo spend lands only where it actually buys you something.

**Primary — Prospeo, off the LinkedIn URL.** No domain needed, so it covers every
keeper including the ones with no company domain:

```bash
curl -s -X POST "https://api.prospeo.io/enrich-person" \
  -H "X-KEY: $PROSPEO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "data": { "linkedin_url": "https://www.linkedin.com/in/janedoe" } }'
# → { "response": { "person": { "email": { "email": "...", "status": "VALID" } },
#                   "company": { "name": "...", "domain": "...", "employee_count": N } } }
```

Read `response.person.email.email` (+ `.status`); `response.company` also returns
domain / employee count / industry you can backfill onto the record. Charged
**1 credit per email found** (misses & errors are free).

**Throttle it.** Prospeo's lower tiers rate-limit hard (HTTP 400
`"Rate limit exceeded"`). Sleep ~9–12s between calls and retry on rate-limit with
backoff — exactly the pattern in the operator's `enrich.py`. Check
`remaining_credits` before the run and warn if the keeper count exceeds it.

**Fallback — Evaboot Email Finder** for Prospeo misses, when the lead has a
domain:

```bash
curl -s -X POST "https://api.evaboot.com/v1/email-finder/" \
  -H "Authorization: Bearer $EVABOOT_API_KEY" -H "Content-Type: application/json" \
  -d '{ "job_name":"gtm-keepers",
        "prospects": [ { "first_name":"Jane","last_name":"Doe",
                         "company_name":"Acme","company_domain":"acme.com" } ] }'
```

Patch the found emails back onto those leads' records (PATCH each lead, or include
`email` on a re-insert). Then point the user at `campaign-writer` for the
ICP-qualified segment.

**Pause for approval before this stage** — it is the variable spend. Show the
keeper count and the credit estimate, and confirm Prospeo has enough credits
(its free tier is ~100/mo and does not roll over).

## How ICP scoring works (and who pays)

1. **The ICP lives in Nous, in the GTM context.** The skill reads yours; it does
   not invent one.
2. **The skill (Claude) does the reasoning**, on the operator's own tokens. Nous
   is never asked to score thousands of leads, so no Nous credits are burned.
3. **Reliability comes from the positive ICP, not the blocklist.** The blocklist
   and Evaboot's match flag are signals that nudge the score; the positive ICP
   definition is what actually judges fit.
4. **Nothing is dropped or auto-deleted.** Every lead is imported and labelled.
   The operator filters ICP vs non-ICP in the list and deletes by hand. A
   misjudged lead is never lost; this is also how you tighten the ICP against
   real pulled data over time.

## Hard rules — never break these

- **Import every lead you extracted.** You paid for it. Never pre-filter leads
  out of the list — the ICP score is the filter, and it lives inside Nous.
- **`Matches Filters: NO` is not off-ICP.** It's a signal, not a gate. Score and
  import those leads too.
- **Blocklist matches company identity with word boundaries, and only lowers the
  score** — it never removes a lead.
- **Structural filters first** in the search; flat positive `OR` keywords only;
  never Boolean `NOT` in the keyword box.
- **Not "Advertising Services"** for modern agencies — it's the legacy trap.
- **Warn at the 2,500 export cap.** Tighten below it for a complete pull.
- **Extract leads-only (`enrich_email:"none"`) first;** emails are Stage 4, on
  ICP-qualified keepers only. Check `/v1/quota/` and surface the extraction cost
  before running.
- **The skill never auto-deletes.** Deletion is the operator's manual control.

## Customize / Set up

- **Tune the structural filters** — years cap, headcount, industries, titles.
- **Set the ICP threshold** — the `icp_score` cutoff (default 40) that flags
  `icp:true` and gates Stage 4 email spend.
- **Edit the blocklist signals** — adjust per niche; they nudge the score.
- **Swap the extractor** — Evaboot is the default; any Sales Nav export that
  returns leads-only works.
- **Swap the email finder** — Prospeo (off the LinkedIn URL) is the default and
  the cheapest; Evaboot Email Finder is the built-in fallback for misses.

## FAQ

**Why import the non-ICP leads instead of dropping them?**
Because you paid to extract every one of them, and `Matches Filters: NO` hides
real ICP leads. Importing all of them, scored and labelled, keeps the list
complete and filterable, costs zero email credits on the non-keepers, and lets
you tighten your ICP against real data. Deletion is your manual choice.

**Why extract without emails first?**
Emails are the variable cost. Finding emails on leads you won't contact is waste.
Extract leads-only, score everyone for free in Nous, and pay for emails only on
ICP-qualified keepers.

**What if I don't have Sales Navigator?**
Use `lead-builder` — same job from Apollo's people search, no Sales Nav needed.

**What does it cost?**
Two providers. **Evaboot** for extraction: 1 credit per lead extracted (deducted
on completion). **Prospeo** for emails: 1 credit per email found off the LinkedIn
URL (misses free, ~$0.02 at volume), Evaboot Email Finder as fallback. Scoring,
dedup, and import are free in Nous. Email credits are spent only on ICP-qualified
net-new keepers.

**Why Prospeo for emails instead of Evaboot?**
It finds the email straight from the LinkedIn URL (no domain needed, so it covers
no-domain keepers too), it's ~2× cheaper per verified email, and it returns
company firmographics you can backfill. Evaboot stays as the fallback on misses.
