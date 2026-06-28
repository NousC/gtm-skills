---
name: lookalike-builder
description: Paste up to 5 companies you'd love to work with (or describe the niche) and AI-Ark's similarity model finds the lookalike companies. The CORE JOB of this skill is building the company table — discover the lookalikes, narrow with include/exclude keywords and firmographics, ICP-score, and save the companies into Nous. The whole search and count is free to preview. It CAN also pull decision makers and BounceBan-verified emails directly, but AI-Ark's people index is thin on small agencies (~37% of 3–20-person companies have an indexed person), so for the people layer the recommended path is to hand the company list to `company-people` (LinkedIn scrape, ~90% coverage). Rule of thumb: lookalike-builder discovers the COMPANIES, company-people turns them into DECISION MAKERS. Use AI-Ark's own people search only on bigger, well-indexed companies (50+) where a title-filtered database query beats scraping a large People tab.
---

# Lookalike lead builder (AI-Ark)

## What it does — and what it's FOR

You paste **up to 5 companies you'd love to work with** — or just describe the
niche — and AI-Ark's similarity model returns the **lookalike companies**. The
**core deliverable of this skill is the company table**: the similar companies,
narrowed by include/exclude keywords + firmographics, ICP-scored, saved into Nous.

That company discovery is the **irreplaceable** thing AI-Ark does — no Sales Nav,
no Apollo UI, one API does lookalike + include + exclude. It's the **only** job we
reach for this skill to do.

> **The split (decided 2026-06-26, proven on a real 527-lead build).** AI-Ark has
> TWO engines and they are NOT equal quality:
> - **Company lookalike search = excellent.** Always use it. This is the company table.
> - **People index = thin for small agencies (~37% of 3–20-person companies had any
>   indexed decision-maker on a real run), dense for bigger ones.**
>
> So **the recommended pipeline is: lookalike-builder for the COMPANIES →
> `company-people` (LinkedIn scrape) for the DECISION MAKERS + emails** (~90%
> coverage on small agencies, because LinkedIn's company page lists nearly every
> founder). Only use AI-Ark's own people search + email-finder on **bigger,
> well-indexed companies (50+)** where a title-filtered database query beats
> scraping a large People tab.

This skill **can** still run the full people + email path end to end (Phases 3–6
below) — keep it for the big-company case, or when the user explicitly wants the
one-tool flow. But by default, **stop at the company table and hand off**.

```
[0] EXCLUDE-OWN auto-build an exclude list from your Nous domains → no re-finds
[1] LOOKALIKE   Company Search lookalikeDomains:[5 seeds]  → similar companies
[2] NARROW      + filters: size, location, INCLUDE keywords → tighter set
[3] EXCLUDE     + exclude keywords + the exclude-own list   → the noise removed
[4] PREVIEW     count + sample, NO emails, NO email spend    → you approve here
[5] SAVE        ICP-score companies → Nous list (the company table)  ◄ DEFAULT STOP
                ─────────────────────────────────────────────────────────────────
                then hand the company list to /company-people for the people layer
                (or, for 50+ companies, continue below with AI-Ark people search)
[6] PEOPLE      (optional) People Search inside those companies → founders
[7] EMAILS      (optional) Track ID → email-finder (BounceBan-verified)
```

The default flow: **paste seeds → AI-Ark similarity → narrow/exclude → preview the
count → save the company table → hand off to `company-people` for decision makers.**

## How to invoke

`/lookalike-builder` — then paste seeds, e.g. *"like anna-agency.com,
bravo-collective.com, growthlab.io — founders, 1–10 people, US"* — or describe
the niche and it builds the spec.

## The demo (this is the video beat)

1. `/lookalike-builder`
2. Paste 5 companies you wish you could clone.
3. It comes back: *"~840 lookalike companies, ~1,100 founders match after your
   excludes. Verified emails would cost ~$X. Want the list?"* — **before any spend.**
4. You tweak (tighten size, add an exclude) or say go.
5. It finds verified emails and a new list lands in your Nous **Lists** — filling
   in live. You never touched a filter UI.

---

## First-run setup (run once as a short interview)

**1. AI-Ark (required) — search + verified emails.** Check for `AIARK_API_KEY`.
Missing → "I find the lookalike companies, the decision makers, and verified
emails through AI-Ark — one API for all of it. `export AIARK_API_KEY=...`
(ai-ark.com → Settings → API). Auth is the `X-TOKEN` header. Plan is usage-based,
credits roll over up to 2x: **$49/mo = 5,000 credits, $79/mo = 15,000 credits**
(verified 2026-06-23). Note the match COUNT is free, but pulling/browsing records
DOES cost credits — this is not search-for-free (see Cost below)."

## Cost (verified 2026-06-23) — read this, AI-Ark meters the search too

AI-Ark credit prices: **valid email found = 0.5**, contact storage = 0.5,
lookalike company browse/store = 0.5, basic company = 0.1, mobile = 5. So a
finished lead (kept contact + verified email) ≈ **1 credit**, and AI-Ark's own
estimator puts **2,000 leads ≈ 2,000 credits**.

- Plans: **$49 = 5,000 credits** (~2.5 runs of 2k), **$79 = 15,000 credits**
  (~7 runs of 2k), credits roll over up to 2x.
- Per-lead at the $79 tier ≈ **$0.005** (15,000 cr / $79, ~1 cr/lead) — much
  cheaper per lead than Apollo at volume.
- ⚠️ **The catch:** browsing lookalike companies to FIND the keepers costs
  0.1–0.5 credits each, on top of the 2,000. The estimator only counts kept
  contacts, so the true cost is "2,000 + however much you paged." Pull the
  **top-N ranked lookalikes** (best matches first), don't page the whole tail.
- vs `lead-builder` (Apollo Basic $59 = 2,500 credits, 1 email = 1 credit, and
  the SEARCH is free): Apollo wins on simplicity + free search at low volume;
  AI-Ark wins on per-lead cost + the semantic lookalike at higher volume.

**2. Nous (required) — dedup + where the list lands.** Check `NOUS_API_KEY`.
Missing → "`export NOUS_API_KEY=pk_xxx` (opennous.cloud → Settings → API keys).
It dedupes against your pipeline and saves the list. Free at opennous.cloud."

No FullEnrich, no MillionVerifier — AI-Ark returns BounceBan-verified emails, so
the verify step is built in.

> ✅ **VERIFIED LIVE 2026-06-23.** Auth (`X-TOKEN`), Company Search, and People
> Search request/response shapes below are confirmed against the real API. The
> ONE thing still to confirm live is the **email-finder request/response** (the
> paid step) — confirm its exact path + verified-status field on the first real
> email run, before scaling. Two behaviours to respect (see notes inline):
> Company Search `totalElements` **caps at 10000** (it's a ranked similarity
> stream, not an exact total), and **firmographic filters are SOFT under
> lookalike** (size 1–10 still returned some 11–50 companies) — enforce size
> strictly as a Claude-side post-filter on `summary.staff.range`.

---

## Phase 1 — Build the targeting spec from the seeds

Turn whatever the user gives into a structured AI-Ark spec. **Do NOT overcomplicate
the keywords.** The lookalike is the engine; a short, sharp **exclude** list plus a
tight **team size** is what separates the real ICP from the look-alikes that share
its vocabulary (a real outbound agency vs a web-design / social / SEO shop that also
says "marketing"). For the common case (outbound / GTM agencies) use the proven
default set below as-is; only tune it if the niche is genuinely different.

```
Seeds (lookalikeDomains, ≤5):  anna-agency.com, bravo-collective.com, growthlab.io
Company filters:
  Employee size:   3–20   (the key lever — Bennet's proven band for agencies)
  Location:        United States, Canada
  INCLUDE keywords: lead generation, demand generation, appointment setting,
                    cold email, outbound, SDR, GTM   (keep it short)
  EXCLUDE keywords: branding, sms, web design, seo, recruiting, staffing,
                    instagram, facebook
People filters:
  Seniority:       founder, c_level
  Titles:          Founder, Co-Founder, CEO, Owner, GTM Engineer, Head of Growth
  Started company: 2015+ (DEFAULT — modern founders; max tenure = thisYear−2015)
Target:            ~1,000
```

### Default exclude set for OUTBOUND / GTM AGENCIES (Bennet's proven defaults 2026-06-23)

`branding, sms, web design, seo, recruiting, staffing, instagram, facebook`
+ **team size 3–20**. This is exactly the spec that produced his working list —
it strips the web/social/SEO/branding/recruiting shops that share the marketing
vocabulary but aren't outbound agencies. **Start here, don't reinvent it.** Add a
term only if the user names a specific kind of noise to remove (e.g. "no PR", "no
agencies that do paid ads"); never pad the list to look thorough.

Ask only for what's genuinely missing (size, geo, buyer title, count) — one
question at a time. **Cross-check against the saved Nous ICP first** (it's the
source of truth — a one-off brief may be a new segment or a slip):

```bash
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

Aligned → say so and move on. Diverges → surface it and ask before spending.

### Exclude what you ALREADY have — automatic, every run (don't pay twice)

**Always do this before the search.** The whole point: you already have leads in
Nous, and you should never re-surface or re-pay for them. So each run, pull every
domain you already own from Nous and feed them to AI-Ark as an exclude list, so
those companies never even appear in the lookalike results.

This is the COMPANY-level guard at search time. (The PERSON-level guard — `/v2/dedup`
by LinkedIn URL right before the paid email step — still runs in Phase 4. Two layers:
exclude known companies up front, dedup known people before paying.)

**Step 1 — collect your existing domains from Nous.** Pull every lead list and
gather each lead's domain (and the domain from its email), into one deduped set:

```bash
# every list, then every lead's domain — the set of companies you already touch
curl -s "https://api.opennous.cloud/api/lead-lists?workspaceId=$WS" \
  -H "Authorization: Bearer $NOUS_API_KEY"   # → list ids
# for each list:
curl -s "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads?workspaceId=$WS" \
  -H "Authorization: Bearer $NOUS_API_KEY"   # → leads[].fields.domain + email domain
```
Collect into a unique, lowercased domain set (strip `www.`, take the part after `@`
for emails). Add any explicit competitor/customer domains the user names.

**Step 2 — build the AI-Ark exclude list from that set.** AI-Ark allows up to 10
exclude lists per search; **its lists expire after 24h**, so rebuild it each run:

```bash
curl -s -X POST "https://api.ai-ark.com/api/developer-portal/v1/save-list" \
  -H "X-TOKEN: $AIARK_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name":"nous-already-have", "items":[ "<domain1>", "<domain2>", … ] }'
# → returns a list id → reference it in the search `lists.company_id.exclude`
```

**Step 3 — reference it in every Company Search** (Phase 2) via
`"lists": { "company_id": { "exclude": ["<EXCLUDE_LIST_UUID>"] } }`. Now the count
and the results already have your existing companies stripped out — the preview
reflects only net-new companies, so you never pay to re-find someone you have.

Tell the user what you excluded: "Excluded the N companies already in your Nous
lists, so this is all net-new." If a domain set is huge (>thousands), it still works;
AI-Ark handles the list, and the Phase 4 person dedup is the backstop either way.

## Phase 2 — Preview the count and a rough cost, then WAIT (no emails yet)

This is the gate the user asked for: **count and approve before paying.** The
search and the count are free of the email cost — emails are the separate paid
step. Run the company search to get the volume, pull a small people sample to
sanity-check the match, and show the estimate. **Do NOT call the email-finder yet.**

```bash
# Lookalike + narrow + exclude. CONFIRMED schema (2026-06-23).
curl -s --max-time 180 -X POST "https://api.ai-ark.com/api/developer-portal/v1/companies" \
  -H "X-TOKEN: $AIARK_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "lookalikeDomains": ["anna-agency.com","bravo-collective.com","growthlab.io"],
    "account": {
      "employeeSize": { "type":"RANGE", "range":[ {"start":3,"end":20} ] },
      "location": { "any": { "include": ["United States","Canada"] } },
      "keyword": {
        "any": {
          "include": { "sources":[ {"mode":"SMART","source":"KEYWORD"}, {"mode":"SMART","source":"DESCRIPTION"} ],
                       "content":["lead generation","outbound","cold email","appointment setting","GTM"] },
          "exclude": { "sources":[ {"mode":"WORD","source":"KEYWORD"}, {"mode":"WORD","source":"DESCRIPTION"} ],
                       "content":["branding","sms","web design","seo","recruiting","staffing","instagram","facebook"] }
        }
      }
    },
    "lists": { "company_id": { "exclude": ["<EXCLUDE_LIST_UUID>"] } },
    "page": 0, "size": 10
  }'
```

Read the count from **`totalElements`** (caps at 10000 = "top similar," not exact).
Each company is in **`content[]`**: domain at `link.domain`, name at
`summary.name`, size at `summary.staff.range` ({start,end}), industry at
`summary.industry`, the company's own keywords at `keywords[]`, LinkedIn at
`link.linkedin`. **Collect `link.domain` for every kept company** — those domains
are what scope the People Search in Phase 3. **Enforce size 1–10 yourself** by
dropping companies whose `summary.staff.range.start` exceeds your cap (the API
filter is soft under lookalike). Match mode reference: `SMART` (fuzzy / semantic),
`WORD` (partial), `STRICT` (exact); keyword `source` ∈ NAME, KEYWORD, SEO,
DESCRIPTION, INDUSTRY.

Then show the user, and wait for an explicit yes:

```
Lookalike from your 3 seeds:
  ~840 similar companies, 1–10 employees, US, after your exclude keywords.
  ~1,100 founders/owners match inside them.

Verified emails (BounceBan) ≈ $X for ~1,100 (charge model confirmed on run).
Search/preview already done — no email spend yet.
Want the list? (or tighten: smaller size band, more excludes, fewer seeds)
```

### Review IN the AI-Ark web app — generate the search link (PREFERRED review path)

The AI-Ark API returns no preview URL, but the **web app encodes the whole search
in a shareable link**, so you can hand the user a link that opens this exact
lookalike + filters + excludes in the AI-Ark UI, where they review, tweak, and
confirm before any spend. Build it from the same spec and present it:

```python
import urllib.parse as u
# seeds as LinkedIn company URLs (preferred) or domains; values joined by '^'
seeds = ["https://www.linkedin.com/company/zevenue",
         "https://www.linkedin.com/company/understory-agency",
         "https://www.linkedin.com/company/slyleadz11"]
regions = ["North America::United States", "North America::Canada"]  # Region::Country
params = {
  "value": "^".join(seeds),
  "employee_size_custom": "3>20",                 # min>max  (caps the soft API size filter)
  "company_hq_include_location_region": "^".join(regions),
  "year_founded": "2015>",                        # founded 2015 or later
  "companies_exclude_keywords": "^".join(["branding","sms","web design","seo","recruiting"]),
}
print("https://app.ai-ark.com/search/company?" + u.urlencode(params, safe="^>:"))
```

Hand it over:

> "Here's the search in AI-Ark, open it and review the companies before any spend:
> <link>. Tweak anything in the UI (size, regions, excludes). When it looks right,
> say go and I'll pull the decision makers and find verified emails."

Encoding rules: multi-value separator is `^`, ranges use `min>max`, "founded after"
is `YEAR>`, region is `Region::Country`, spaces become `+`. **Keep the link and the
API call in sync** — they're the same spec, so when the user edits the link, read
the new params back and mirror them into the Phase 2 API body before running.

> Note: the web link supports a `year_founded` filter directly; if the API Company
> Search body has no founded-year field, post-filter on `summary.founded_year >= 2015`
> in Claude (same way you enforce size). The web link is the review; the API is the run.

### (Fallback) stage a no-email list in Nous for review

If the user would rather review inside Nous, create the Nous list now WITHOUT
emails, drop the people in, and hand them that link instead. Same dedup/score
later, emails deferred until they approve.

## Phase 3 — Pull the decision makers (People Search → Track ID)

On approval, pull the decision makers **inside the companies from Phase 2**.
People Search does NOT take `lookalikeDomains` — you scope it by feeding the
**company domains you collected** into `account.domain.any.include`. It returns a
**`trackId`** that the email-finder uses next. CONFIRMED schema (2026-06-23):

```bash
curl -s --max-time 180 -X POST "https://api.ai-ark.com/api/developer-portal/v1/people" \
  -H "X-TOKEN: $AIARK_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "account": {
      "domain": { "any": { "include": ["acme.com","bluecraftleads.com","winningcontacts.com"] } }
    },
    "contact": {
      "seniority": { "any": { "include": ["founder","c_level"], "exclude": ["intern","individual_contributor"] } },
      "experience": { "current": {
        "title": { "any": { "include": { "mode":"SMART",
                     "content":["Founder","Co-Founder","CEO","Owner","Managing Partner",
                                "GTM Engineer","Growth Engineer","Head of Growth","Head of GTM"] } } },
        "duration": { "currentJob": { "max": { "year": 11, "month": 0 } } } } },
      "keyword": { "any": { "exclude": { "sources":[ {"mode":"WORD","source":"HEADLINE"} ],
                                         "content":["assistant","freelance"] } } }
    },
    "lists": { "people_id": { "exclude": ["<EXCLUDE_PEOPLE_LIST_UUID>"] } },
    "page": 0, "size": 100
  }'
```

Response: people in **`content[]`** — name at `profile.full_name`
(`profile.first_name` / `profile.last_name`), title at `profile.title`, headline
at `profile.headline`, LinkedIn at `link.linkedin`, company at
`position_groups[0].company.name`. The **`trackId`** is top-level (expires ~6h —
do the email step in the same run). `seniority` enums: `founder, c_level, head,
senior, manager, individual_contributor, intern`. **Domain include is capped per
request — batch the company domains** (e.g. 25–50 per call) and page each batch.

> 💡 DEFAULT: started current company **2015 or later** (modern founders, not
> 15–20-year legacy veterans). It's a *duration* lever
> (`contact.experience.current.duration.currentJob.max:{year,month}`), so set
> `max.year = currentYear − 2015` — **11 as of 2026** (recompute from today's year
> each run so it always anchors to 2015, not a drifting fixed number). Loosen or
> drop it only if the niche genuinely wants long-tenure operators.

## Phase 4 — Dedup vs Nous (free), then verified emails (the paid step)

**Dedup first** so you never buy an email for someone you already own. AI-Ark
returns each person's LinkedIn URL — dedup on it:

```bash
curl -s -X POST "https://api.opennous.cloud/v2/dedup" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "linkedin_urls": ["https://www.linkedin.com/in/jane-doe", "..."] }'
```

Route each person: **`net_new`** → find email (you pay). **`needs_enrichment`**
(own them, stale >90d) → add to list + re-enrich via Nous, don't re-buy.
**`reusable`** (fresh verified email already) → add to list, skip the finder.
**`engaged`/`recent`** → skip, don't cold-touch mid-conversation.

**The email-finder is async, and the RELIABLE way to collect results is to POLL
`/inquiries` and write each email as it completes** (proven end to end 2026-06-23).
Submit requires a `webhook` param (a bare submit fails `"webhook is required"`),
so include one — but **do NOT depend on the webhook push**: AI-Ark only fires it
when the WHOLE job is DONE, and a single stuck lead blocks the entire batch (seen
live — 4 emails ready, webhook never fired). The webhook receiver Nous hosts is a
backup for clean jobs; the poll is the primary path.

**Order:** create the Nous list + insert the leads FIRST (Phase 5), then submit,
then poll + write.

```bash
# 1. Submit (trackId in BODY, webhook required — point it at the Nous backup receiver).
curl -s -X POST "https://api.ai-ark.com/api/developer-portal/v1/people/email-finder" \
  -H "X-TOKEN: $AIARK_API_KEY" -H "Content-Type: application/json" \
  -d '{ "trackId": "<TRACK_ID>",
        "webhook": "https://api.opennous.cloud/inbound/aiark/<WORKSPACE_ID>/<LIST_ID>" }'

# 2. POLL until every inquiry is state:"DONE" (per-lead can take 1-3 min). Each:
#    input{firstname,lastname,domain} + output[]{address,status,domainType,found}.
curl -s "https://api.ai-ark.com/api/developer-portal/v1/people/email-finder/<TRACK_ID>/inquiries?page=0&size=100" \
  -H "X-TOKEN: $AIARK_API_KEY"

# 3. For each DONE inquiry with output.found==true and status VALID (or safe
#    CATCH_ALL), MATCH it to its lead by name and PATCH the email onto the lead.
#    The list fills live as you poll; write incrementally, don't wait for all.
curl -s -X PATCH "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads/<LEAD_ID>" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "workspaceId":"<WS>", "key":"email", "value":"jane@acme.com" }'
```

Polling loop: every ~20-30s, pull `/inquiries`, write any newly-DONE found emails,
stop when all inquiries are DONE or a sensible timeout (~10 min for a big batch).
Match each result to its lead by **full name** (`firstname + lastname` ↔ the lead's
name; the leads were inserted with `profile.full_name`). Keep `found:true` +
status `VALID`; treat `CATCH_ALL` as riskier (keep but flag). `AI-Ark's JSON can
contain raw control chars` — parse leniently (`json.loads(..., strict=False)`).

> Backup only: the Nous webhook receiver `apps/worker/src/webhooks/handlers/aiark.mjs`
> (route `/inbound/aiark/:workspaceId/:leadListId`, deployed) sets emails by
> name+domain match if AI-Ark ever does push a clean completed job. The poll above
> is what the skill relies on.

## Phase 5 — Save to a Nous lead list (stream in live)

Create the list, declare columns, insert leads as their emails verify so the list
fills in live. Reuse the proven `lead-builder` / `sales-nav-builder` pattern:

```bash
# 1. create — response carries id AND workspace_id:
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name":"Lookalike · GTM agency founders · 1–10 · US", "source":"lookalike_ai_ark" }'
# → { "lead_list": { "id":"<LIST_ID>", "workspace_id":"<WS>", ... } }

# 2. declare columns (PATCH REQUIRES workspaceId in body):
curl -s -X PATCH "https://api.opennous.cloud/api/lead-lists/<LIST_ID>" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "workspaceId":"<WS>", "columns":[
        {"key":"icp_score","label":"ICP"}, {"key":"title","label":"Title"},
        {"key":"niche","label":"Niche"}, {"key":"matched_on","label":"Match"} ] }'

# 3. insert leads (source stamps the system Source column on every lead):
curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "source":"Lookalike (AI-Ark)", "importDuplicates": true, "leads":[
        { "name":"Jane Doe", "email":"jane@acme.com", "company":"Acme",
          "linkedin_url":"https://www.linkedin.com/in/janedoe",
          "fields": { "title":"Founder", "domain":"acme.com",
                      "industry":"agency", "employee_count": 8,
                      "niche":"outbound GTM agency", "matched_on":"lookalike",
                      "icp_score": 86, "source":"lookalike_ai_ark" } } ] }'
```

Map from AI-Ark people fields: `name` ← `profile.full_name`, `linkedin_url` ←
`link.linkedin`, `company` ← `position_groups[0].company.name`, `fields.title` ←
`profile.title`, `email` ← the email-finder result. Dedup (Phase 4) keys on
`link.linkedin`.

**Always pass firmographics** — `fields.industry` ← `summary.industry` (the
AI-Ark vertical, e.g. "agency") and `fields.employee_count` ← `summary.staff.range.start`
(the size band you filtered on). These become company firmographic CLAIMS on
ingest and are what the ICP scorecard scores on — without them the lead can't be
scored. You already have both from the company search, so never drop them.

Optionally **ICP-score every lead** against the Nous GTM profile before insert
(same rubric as `sales-nav-builder` — score everything, drop nothing, the score
is the filter inside Nous). Then hand the user the Nous list link and point them
at `signal-scan` / `content-scan` next, then `campaign-writer`.

> Resolution is async: ~1,200 leads take a few minutes; reading back immediately
> shows `status:"pending"` — that's mid-resolution, not a failure.

## The credit gate — the most important rule (never skip)

AI-Ark charges credits for **pulling records, not just emails** (contact 0.5 +
email 0.5 ≈ 1 credit per finished lead). So the spend starts the moment you pull
people, not just at the email step. The gate:

1. Run the People Search and read **only the count** (`totalElements`) — this is
   effectively free (a size:1 call). Do NOT page or pull the full set yet.
2. **Stop and tell the user the real number, every time:**
   > "Found ~4,000 founders / GTM people across your 2,100 companies. Pulling them
   > with verified emails ≈ **4,000 credits (~$21)**. You have N credits left.
   > Proceed, or tighten the filter?"
   Compute it: `kept_people × 1 credit` (0.5 pull + 0.5 email), minus whatever
   Nous dedup already owns. State the credit number AND the rough dollars.
3. **Only on an explicit yes** do you pull the records + run the email-finder.
   No confirmation = spend nothing. Never pull the full set or find emails before
   the user says go.

## Hard rules — never break these

- **Confirm the credit cost before ANY paid pull.** Get the count free, show
  "X people ≈ Y credits", wait for an explicit yes. Pulling records AND finding
  emails both cost — the gate covers both, not just emails.
- **Exclude what you already have, EVERY run.** Before the search, build the
  exclude-own list from your Nous domains (the "Exclude what you already have" step)
  and pass it in `lists.company_id.exclude`, so existing companies never re-surface.
- **Dedup before you reveal.** Reveal emails only on `net_new`. Re-enrich
  `needs_enrichment`, reuse `reusable` — never re-buy a person you already own.
- **Confirm field names live on the first run** (filters, ids, Track ID, email
  status) before trusting the calls at scale. Read one raw response first.
- **Never invent an email.** AI-Ark returns BounceBan-verified emails; keep only
  the verified/safe ones, flag the rest — don't guess.
- **Stamp the lead source** (`"source":"Lookalike (AI-Ark)"`) so reply rates can
  be compared across sources (this vs Apollo lead-builder vs Sales Nav vs inbound).
- **Name the list from the ICP** and give the user the link.
- **Rebuild exclude lists each run** — AI-Ark lists expire after 24h.

## Customize / Set up

- **Seed up to 5** companies (domains or LinkedIn URLs) — the similarity engine.
- **Tune the spec** — size, location, include/exclude keywords, titles, seniority.
- **Exclude whole lists** — customers, competitors, an industry block (≤10 lists).
- **Defer emails for review** — stage the no-email list in Nous, approve, then find.
- **ICP threshold** — set the `icp_score` cutoff that flags keepers.

## FAQ

**Can I see how many leads before paying?** The match COUNT (`totalElements`) is
free to read. But pulling/browsing the actual records costs credits (companies
0.1–0.5 each), so the count is free, the list-building is not. Phase 2 shows the
volume; the bigger spend is the verified emails (0.5 credit each) after approval.

**Can I review and edit the list before emails are found?** Yes — the optional
Phase 2 stage drops the no-email candidates into a Nous list and gives you the
link. Remove anyone, then approve and only the survivors get verified emails.

**How is this different from `lead-builder`?** `lead-builder` uses Apollo, whose
API can't exclude keywords/titles (so it needs the Apollo UI). This uses AI-Ark,
whose API does lookalike + include + **exclude** + **verified emails** natively —
no UI, no second verifier, fewer tools. Use `lead-builder` if you already pay for
Apollo and don't want a new vendor; use this for the cleaner, fully-automated path.

**Do I need a verifier like MillionVerifier?** No. AI-Ark verifies emails in real
time via BounceBan, so verification is built into the email-finder step.

**What does it cost?** (verified 2026-06-23) AI-Ark is usage-based, credits roll
over up to 2x: **$49/mo = 5,000 credits, $79/mo = 15,000 credits**. Credit prices:
valid email = 0.5, contact storage = 0.5, lookalike company browse = 0.5, basic
company = 0.1, mobile = 5. A finished lead ≈ 1 credit, and their estimator puts
**2,000 leads ≈ 2,000 credits** — but that ignores the company-browsing to find
them (0.1–0.5 each), which is real and is what drains a free account fast. At the
$79 tier per-lead ≈ $0.005. Pull the top-N ranked lookalikes, not the whole tail.
Nous dedup keeps you from paying for an email twice.
