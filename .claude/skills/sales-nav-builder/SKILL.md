---
name: sales-nav-builder
description: Turn a LinkedIn Sales Navigator search into a clean, ICP-scored lead list in Nous, paying for emails only on the leads you keep. Claude builds the search from structural filters first (years at company, headcount, the right industries) with modern-vocabulary keywords second, you run it, and Evaboot extracts the leads with no emails. Nous then excludes the wrong company types, dedupes, and ICP-scores every lead. Only the ICP-qualified net-new leads get emails found, so the variable cost is spent on keepers. Non-ICP leads stay in the list labelled, never deleted. For people who have Sales Navigator.
---

# Sales Nav lead builder

## What it does

You have **LinkedIn Sales Navigator** — the sharpest targeting filters that
exist. This skill turns a Sales Nav search into a clean, **ICP-scored** lead
list in Nous, and spends email credits **only on the leads you actually keep**.

The money model is the whole point. Finding and verifying emails is the variable
cost, so we never spend it on junk company types or duplicates. We pay to pull
the LinkedIn data, **filter and score for free inside Nous**, then pay to find
emails only on the survivors.

```
Stage 1  EXTRACT (pay)   Evaboot, NO emails → LinkedIn data only.   1 credit / lead
Stage 2  FILTER (free)   In Nous: exclude wrong company types → dedup → ICP-score
Stage 3  EMAILS (pay)    Evaboot Email Finder, only on ICP-qualified net-new.
                         1 credit / email found (misses are free)
Stage 4  SAVE            Lead list, every lead tagged icp true/false, emails on
                         the qualified ones, non-ICP kept and labelled
```

The flow: **Claude (structural search) → Sales Nav → Evaboot extract (no emails)
→ Nous (exclude + dedup + score) → Evaboot emails on keepers → Nous list.**

## How to invoke

`/sales-nav-builder` — or *"build me a list of outbound GTM agency founders from
Sales Navigator."*

## First-run setup (you, the agent, run this once as a short interview)

**1. LinkedIn Sales Navigator (required).** Confirm the user has a seat. Evaboot
extracts live from their own Sales Nav search (no third-party database, which
keeps it account-safe and GDPR-clean). No Sales Nav → point them to
`lead-builder`.

**2. Evaboot (required) — extract + email finder.** Check for `EVABOOT_API_KEY`
(dashboard → Integrations → API). It's credit-based: **1 credit to export a
lead (no email), 1 more only when an email is found** (misses are free). From
$9/mo.

**3. Nous — the ICP, dedup, and where the list lands (required).** Check
`NOUS_API_KEY` (`pk_`), and connect the MCP for `get_gtm_profile`. Nous holds the
**ICP / GTM context** the skill scores against, dedups the leads, and stores the
list. The scoring itself runs in the skill (your Claude tokens), not on Nous.

---

## Phase 1 — Build the search: structural filters first, keywords second

The biggest mistake is leading with keywords. Legacy local agencies and modern
AI-native GTM agencies **both** say "lead generation" and "demand generation" —
keywords can't separate them. **Structure can.** Build the search in this order:

**1. Years in current company ≤ 5 — the single biggest lever.** This instantly
removes the 16-to-24-year-tenure legacy marketing, SEO, and design veterans and
keeps the founders who started modern agencies. Lead with this.

**2. Company headcount** — the size band from the saved ICP (e.g. 1–10).

**3. The right industries — and the trap to avoid.** For modern AI-native GTM
agencies, do **not** use **"Advertising Services"** — that is where the
word-of-mouth, SEO, web-design, and branding shops live. The actual ICP founders
are tagged under:
> **Marketing Services · Business Consulting & Services · IT Services & IT
> Consulting · Software Development · Technology, Information & Internet**

**4. Seniority / title** — Founder, Co-founder, CEO, Owner.

**5. Keywords — a FLAT POSITIVE `OR` list, modern vocabulary only.** This is the
precision lever once structure is doing the separating. Use terms a 20-year
word-of-mouth shop never uses:
> `Clay OR Claude OR "AI-native" OR GTM OR "go-to-market" OR "AI SDR" OR RevOps
> OR outbound OR "cold email"`

Keep generic `lead generation` / `demand generation` only when the structural
filters above are already separating legacy from modern.

### Hard keyword rule — never break this

Sales Nav's keyword box **breaks on Boolean `NOT`.** A long `(...) AND NOT (...)`
returns **0 results** every time, and even a short `(positives) NOT (negatives)`
returns 0 when a leading paren is dropped on paste, a smart-quote sneaks in, or
the box hits its length limit. **Never put exclusions in the keyword box.** Use a
flat positive `OR` list only, and do *all* exclusion downstream in Stage 2.

### The 2,500 export cap

Sales Nav silently truncates any export at **2,500 leads** — a search returning
3,000 loses 500 with no warning. If the preview count is above 2,500, tell the
user to **tighten below it** (narrow the years band, headcount, or geography) so
the pull is complete.

### Cross-check the saved ICP

```bash
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

Aligned → say so. Diverges → surface it and ask before running.

## Phase 2 — Confirm the search and the two-stage cost, then run

Show the search back, the volume (Sales Nav's result count, under 2,500), and the
**two-stage** cost — making clear emails are charged only on survivors.

```
Sales Nav search:
  Years in company:  ≤ 5
  Headcount:         1–10
  Industries:        Marketing Services, Business Consulting, IT Services,
                     Software Development, Technology & Internet
                     (NOT Advertising Services)
  Titles:            Founder / Co-founder / CEO / Owner
  Keywords (flat OR): Clay OR Claude OR "AI-native" OR GTM OR "AI SDR" OR RevOps
  Geo:               United States

Sales Nav shows ~1,300 results (under the 2,500 cap — complete pull).

Stage 1 extract:  ~1,300 leads × 1 credit            = ~1,300 credits  (now)
Stage 2 filter:   exclude wrong types + dedup + score = free
Stage 3 emails:   ~850 ICP net-new × ~70% hit         = ~600 credits   (only on keepers)
Total: ~1,900 credits  (vs ~2,200 to find emails on all 1,300 — and cleaner)
Run it?
```

## Phase 3 — Extract LEADS ONLY (no emails), the credit-saver

Default the export to **No Emails** (leads-only) — this is the documented
default, because emails are Stage 3. Prefer Evaboot's **LinkedIn Extraction API**
endpoint with the Sales Nav search URL and emails off; if you don't have API
extraction wired, the user runs the search in Sales Nav and clicks **Export with
Evaboot** with email-finding **off**, then hands you the leads-only file.

Either way you get, per lead: `name`, `title`, `company`, `company_domain`,
`linkedin_url` — **no email yet**. Cost: 1 credit per lead.

## Phase 3.5 — Filter and score against the ICP (no Evaboot or Nous credits)

This whole stage runs **inside the skill** — you (Claude) read the workspace's
ICP and reason over every lead. It spends **no Evaboot credits and no Nous
credits**; the only cost is the operator's own Claude tokens. That is by design:
**Nous holds the ICP, the skill does the reasoning.** We never burn Nous credits
scoring thousands of leads. (See "How ICP scoring works" below.)

Do all three before spending a single email credit.

### 1. Pull the full ICP from the Nous GTM context (the source of truth)

The exclusion list is a fast first pass, but the **real** judge is the
workspace's own ICP model — already built and maintained in the GTM context.
Fetch it first, and use the *whole* thing (who we target, the firmographics, the
signals/vocabulary, the disqualifiers):

```bash
# the GTM profile / ICP — same data as the get_gtm_profile MCP tool
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

If the Nous MCP is connected, prefer the `get_gtm_profile` tool — it returns the
ICP scorecard and context directly. **This profile is what makes scoring
reliable for company types the exclusion list never anticipated** (see step 3).

### 2. Exclude the obvious wrong company types (fast deterministic pass)

Drop leads whose **company name or description** centres on a non-ICP business.
Match on the company's *identity*, not a stray profile mention, so a real GTM
agency that says "marketing" once isn't nuked. This default list is **editable**
— it is a convenience, not the source of truth:

> design · web design · web development · website · graphic · UX/UI · SEO ·
> search engine · branding · logo · creative · recruiting · staffing ·
> recruitment · talent · headhunting · executive search · word of mouth ·
> PR · public relations · social media management · digital marketing

### 3. Dedup by domain — Nous (free)

```bash
curl -s -X POST "https://api.opennous.cloud/v2/dedup" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "domains": ["acme.com","beta.io"] }'
```

Keep `status === 'net_new'` — no point paying for emails on leads already in the
pipeline.

### 4. ICP-score every surviving lead against the GTM profile

For each remaining lead, judge its **company** against the ICP you pulled in step
1 and write three values into the lead's `fields`:

- `icp`: `true | false` — does this company match the ICP?
- `icp_score`: `0–100` — how strong the fit is.
- `icp_reason`: one short sentence — *why*, citing what matched or missed.

Score against the **positive ICP definition**, not just the blocklist. A company
that isn't on the exclusion list but clearly isn't a GTM agency (say an event
agency, a tax firm) still scores low because it doesn't match the profile. That
is the reliability: **the blocklist catches the known junk, the ICP profile
catches everything else.** Only `icp: true` leads proceed to Stage 3 emails;
`icp: false` leads are kept in the list, labelled with their score and reason.

## Phase 4 — Find emails on keepers, save the ICP-segmented list

### Find emails — Evaboot Email Finder, ICP-qualified net-new only

```bash
curl -s -X POST "https://api.evaboot.com/v1/email-finder/" \
  -H "Authorization: Bearer $EVABOOT_API_KEY" -H "Content-Type: application/json" \
  -d '{ "prospects": [
        { "first_name":"Jane","last_name":"Doe",
          "company_name":"Acme","company_domain":"acme.com" } ] }'
```

Charged 1 credit per email **found** (misses are free). Hit rate runs ~60–80%.

### Save the list — every lead tagged, non-ICP kept

Create the list, declare the `icp` columns so they show and filter in the Nous
UI, then insert **all** surviving net-new leads. ICP-qualified leads carry their
found email; non-ICP leads are saved **leads-only, no email spend**, flagged:

```bash
# 1. create the list — the response carries BOTH the id AND the workspace_id:
#    { "lead_list": { "id": "<LIST_ID>", "workspace_id": "<WS>", ... } }
#    Capture both. The API key alone authorizes this (no workspaceId in body).
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name": "Founder · GTM agencies · 1–10 · US", "source": "sales_nav" }'

# 2. declare the display columns (icp + icp_score → filterable ICP vs non-ICP).
#    This PATCH REQUIRES workspaceId in the body — use the <WS> from step 1.
curl -s -X PATCH "https://api.opennous.cloud/api/lead-lists/<LIST_ID>" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "workspaceId": "<WS>",
        "columns": [ {"key":"title","label":"Title"},
                     {"key":"icp","label":"ICP"},
                     {"key":"icp_score","label":"ICP score"},
                     {"key":"icp_reason","label":"Why"} ] }'

# 3. insert every surviving net-new lead (email only on the ICP-qualified ones).
#    The API key authorizes this — no workspaceId needed in the body.
curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "leads": [
        { "name":"Jane Doe", "email":"jane@acme.com",
          "linkedin_url":"https://www.linkedin.com/in/janedoe", "company":"Acme",
          "fields": { "title":"Founder", "icp": true, "icp_score": 86,
                      "icp_reason":"AI-native GTM agency on Clay, 1–10, US — core ICP",
                      "source":"Sales Nav lead builder", "enriched_by":"evaboot" } },
        { "name":"Bob Legacy", "company":"OldSEO Co",
          "linkedin_url":"https://www.linkedin.com/in/boblegacy",
          "fields": { "title":"Owner", "icp": false, "icp_score": 22,
                      "icp_reason":"SEO/web-design shop, not a GTM agency",
                      "source":"Sales Nav lead builder" } } ] }'
```

The list is now filterable **ICP vs non-ICP** — you see the qualified core with
emails, and the rest you also pulled, leads-only, with a score and a reason,
without losing them. In the Nous list the operator can review the non-ICP rows
and **delete** any that are truly junk, or keep one that was misjudged. That
keeps the ICP definition inspectable and tightenable against real pulled data.

**Expect a short delay.** Leads land immediately, but Nous resolves them in the
background, so names and the `icp` tags fill in **a few seconds (~10s) after the
insert** — tell the operator the list will populate shortly, don't report it as
empty. Then point the user at `campaign-writer` for the ICP-qualified segment.

## How ICP scoring works (and who pays) — read this

Full transparency on the mechanism, because it is the heart of the skill:

1. **The ICP lives in Nous, in the GTM context.** Nous already holds the full
   ICP model — firmographics, target signals, vocabulary, disqualifiers — built
   and maintained on the GTM Context page. The skill does **not** invent an ICP;
   it reads yours (`get_gtm_profile` / `GET /v2/workspace/facts?categories=ICP,Market`).
2. **The skill (Claude) does the reasoning.** Scoring each lead against that ICP
   is an LLM judgment, and it runs **in the skill, on the operator's own Claude
   tokens.** Nous is never asked to score thousands of leads, so **we never burn
   Nous credits on it.** The split is deliberate: *Nous stores the ICP, the skill
   reasons with it.*
3. **Reliability comes from the positive ICP, not the blocklist.** The exclusion
   list is a fast first pass for known junk (SEO, design, recruiting shops). Every
   other company is judged against the *positive* ICP definition, so a company
   type nobody listed still gets a correct low score when it doesn't fit. The
   blocklist handles the obvious; the ICP profile handles the unknown.
4. **Nothing is deleted automatically — it is a control step.** Every lead is
   kept and labelled `icp` / `icp_score` / `icp_reason`. The operator reviews in
   the Nous list, filters ICP vs non-ICP, and deletes rows by hand. A misjudged
   lead is never lost; a confirmed-junk lead can be removed.

This same logic applies to **any bulk set** the skill scores — it reads the GTM
profile once, then scores the whole batch against it before any email spend.

## Hard rules — never break these

- **Structural filters first.** Years-in-company ≤ 5, headcount, the right
  industries — keywords are secondary.
- **Flat positive `OR` keywords only.** Never put exclusions in the Sales Nav
  keyword box; it returns 0 results. All exclusion happens in Stage 2.
- **Not "Advertising Services"** for modern agencies — it's the legacy trap.
- **Warn at the 2,500 export cap.** Tighten below it for a complete pull.
- **Extract leads-only first.** Emails are Stage 3, on keepers only — never pay
  to find emails on junk types or duplicates.
- **Score against the GTM profile, not a guess.** Pull the ICP from Nous first
  (`get_gtm_profile`); the blocklist is only a first pass.
- **The skill never auto-deletes.** It keeps and labels every lead (`icp` /
  `icp_score` / `icp_reason`); deletion is the operator's manual control step in
  the Nous list, so a misjudged lead is never lost.
- **Stamp the lead source.** Highly recommended: set `fields.source` to
  `"Sales Nav lead builder"` on every lead, so reply rates can be compared across
  lead sources later (this skill vs Apollo vs inbound vs LinkedIn engagers).

## Customize / Set up

- **Tune the structural filters** — the years cap, headcount band, industries,
  titles.
- **Edit the exclusion list** — it's the default above, adjust per niche.
- **Set the ICP threshold** — the `icp_score` cutoff that gates email spend.
- **Swap the extractor** — Evaboot is the default; any Sales Nav export tool that
  returns leads-only works.

## Frequently asked questions

**Why extract without emails first?**
Emails are the variable cost. Pulling LinkedIn data is unavoidable, but finding
emails on junk company types or duplicates is pure waste. So we extract leads
only, filter and score for free in Nous, and pay for emails on survivors. On a
~1,300-lead run that's ~1,900 credits instead of ~2,200, and a cleaner list.

**Why keep non-ICP leads instead of deleting them?**
So the list stays filterable and you keep visibility into what you pulled. It
also lets you inspect and tighten your ICP definition against real data over
time. Non-ICP leads cost no email credits — they sit leads-only and flagged.

**What if I don't have Sales Navigator?**
Use `lead-builder` — same job from Apollo's people search with Claude building
the targeting, no Sales Nav needed.

**What does it cost?**
Sales Nav (which you have) plus Evaboot credits: 1 per lead extracted, 1 per
email found (misses free). The exclusion, dedup, and ICP scoring are free in
Nous, and email credits are spent only on ICP-qualified net-new leads.
