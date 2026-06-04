---
name: sales-nav-builder
description: Turn a LinkedIn Sales Navigator search into a clean, verified lead list in Nous. Claude expands your ICP into the exact Sales Nav filters and keywords to apply, you run the search, and Evaboot extracts the leads and finds plus verifies every work email in one pass. It dedupes against Nous and saves the list, ready for outbound. The sharpest targeting available, for people who have Sales Navigator. Pairs with lead-builder, which is the no-Sales-Nav path.
---

# Sales Nav lead builder

## What it does

You have **LinkedIn Sales Navigator** — the best targeting filters that exist.
This skill turns a Sales Nav search into a finished lead list: **Claude builds
the search**, **Evaboot extracts it and verifies every email**, and the leads
land **deduped in a Nous lead list** ready for outbound.

The division of labor is the point. Sales Nav has unbeatable filters but **won't
let you export**. Evaboot does the one hard, fragile thing — pulling the results
out and verifying the emails — as a maintained tool, safely. The skill does what
an LLM is good at: turning your ICP into a sharp search, and the Nous
integration.

The flow: **Claude (search spec) → Sales Nav (the search) → Evaboot (extract +
verify email) → Nous (dedup + save).**

## How to invoke

`/sales-nav-builder` — or *"build me a list of outbound agency founders from
Sales Navigator."*

## First-run setup (you, the agent, run this once as a short interview)

**1. LinkedIn Sales Navigator (required).** Confirm the user has a Sales Nav
seat. Evaboot will not run without it — it extracts live from your own Sales Nav
search (no third-party database, which is what keeps it GDPR-clean and your
account safe). No Sales Nav → point them to `lead-builder` instead.

**2. Evaboot (required) — extract + find email + verify.** Check for
`EVABOOT_API_KEY`. Missing → "Evaboot pulls your Sales Nav search out and finds
plus verifies every work email. Grab a key at evaboot.com → settings → API, then
`export EVABOOT_API_KEY=...`. It's credit-based — a lead with a verified email is
2 credits — from $9/mo."

**3. Nous — dedup + where the list lands (required).** Check `NOUS_API_KEY`.
Missing → "`export NOUS_API_KEY=pk_xxx` (opennous.cloud → Settings → API keys).
It dedupes against your pipeline and saves the list. Free at opennous.cloud."

---

## Phase 1 — Build the Sales Nav search (Claude's job)

Sales Nav is only as good as the search you run. Turn the user's ICP into the
exact filters and keyword set to apply — the same targeting intelligence as
`lead-builder`, aimed at Sales Nav's filter panel:

- **Titles / seniority** → Founder, Co-founder, CEO, Owner (and the seniority
  toggle).
- **Company headcount** → the size band (e.g. Self-employed + 1–10).
- **Industry** → the closest Sales Nav industry (e.g. Marketing & Advertising).
- **Keywords** → expand the category into every variant and give the user the
  exact keyword string to paste — *"lead generation OR demand generation OR
  appointment setting OR cold email OR outbound OR SDR OR RevOps."*
- **Geography** → region / country.

Then either:
- **Hand the user the filter recipe** to apply in Sales Nav and copy the search
  URL back, or
- If they already have a Sales Nav search URL, take it directly.

### Cross-check against the saved ICP

```bash
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

Aligned → say so. Diverges → surface it and ask before running, same as
`lead-builder`.

## Phase 2 — Confirm the search and the credit cost, then run

Show the search back, the expected volume (Sales Nav shows the result count), and
the Evaboot credit cost. **Wait for a yes.**

```
Sales Nav search:
  Titles:   Founder / Co-founder / CEO / Owner
  Size:     1–10 employees
  Industry: Marketing & Advertising
  Keywords: lead generation OR demand generation OR appointment setting OR
            cold email OR outbound OR SDR
  Geo:      United States

Sales Nav shows ~2,400 results. After dedup against Nous, ~1,800 net-new.
Evaboot cost: ~1,800 leads × 2 credits = ~3,600 credits.
Run it?
```

## Phase 3 — Extract, verify, save to Nous, report back

On a yes, run Evaboot on the search, dedup, and save the list:

> "Started. About 1,800 verified leads will land in your new Nous list
> **'Founder · outbound agencies · 1–10 · US'**. I'll dedup against your pipeline
> as they come in."

**Default list title:** `<Buyer> · <niche> · <size> · <geo>`.

---

## The pipeline (the real calls)

Evaboot splits into two jobs: the **Chrome extension** extracts the leads out of
your Sales Nav search, and the **API** finds + verifies the emails. The skill
uses both.

### 1a. Extract the leads — Evaboot Chrome extension (you, once per search)

Sales Nav won't export, and Evaboot's *extraction* lives in its extension, not
the API. So: run the search in Sales Nav → click **Export with Evaboot** → it
cleans the list. Turn email-finding **on** in the export and it returns the
finished file in one pass; leave it off to export leads-only and let the API do
emails in 1b. Hand this skill the resulting CSV (name, title, company, domain,
LinkedIn URL, and — if enabled — a verified email).

### 1b. Find + verify the emails — Evaboot API (only if the CSV has no emails)

For a leads-only export, find and verify each email through the confirmed API
endpoint (1 credit per email found + verified; Bearer token from the dashboard →
Integrations → API):

```bash
curl -s -X POST "https://api.evaboot.com/v1/email-finder/" \
  -H "Authorization: Bearer $EVABOOT_API_KEY" -H "Content-Type: application/json" \
  -d '{ "prospects": [
        { "first_name":"Jane","last_name":"Doe",
          "company_name":"Acme","domain":"acme.com" } ] }'
```

Returns the work email with a verification status; keep the verified ones.
Evaboot is API + extension only (no MCP) — the extension does the Sales Nav
extraction, this endpoint does the emails.

### 2. Dedup by domain — Nous (free)

```bash
curl -s -X POST "https://api.opennous.cloud/v2/dedup" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "domains": ["acme.com","beta.io"] }'
```

Keep `status === 'net_new'`.

### 3. Save to a Nous lead list

```bash
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name": "Founder · outbound agencies · 1–10 · US", "source": "sales_nav" }'

curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "leads": [
        { "name":"Jane Doe", "email":"jane@acme.com",
          "linkedin_url":"https://www.linkedin.com/in/janedoe", "company":"Acme",
          "fields": { "title":"Founder", "source":"sales_nav",
                      "enriched_by":"evaboot" } } ] }'
```

Then point the user at `campaign-writer`.

## Hard rules — never break these

- **Build the search first.** A sharp Sales Nav search is the product — expand
  the category into real keyword variants, don't run "agencies."
- **Cost and approval before you run.** Phase 2 shows the credit estimate.
- **Dedup before you keep.** Run `/v2/dedup`; keep `net_new`.
- **Keep only verified emails.** Evaboot verifies; never save an unverified guess.
- **Name the list from the ICP.**

## Customize / Set up

- **Two ways in** — let Claude build the search recipe, or hand it an existing
  Sales Nav URL.
- **Swap the extractor** — Evaboot is the default; the same flow works with any
  Sales Nav export tool that returns verified emails (Wiza, etc.) if you set its
  key instead.
- **Rename the default list.**

## Frequently asked questions

**Why Sales Nav + Evaboot instead of just a database?**
Sales Nav's filters are sharper than any API's, especially for small, niche
companies, and the data is live. Evaboot is the cleanest way to get it out with
verified emails — one tool, account-safe, from $9/mo.

**What if I don't have Sales Navigator?**
Use `lead-builder` — it does the same job from Apollo's free people search with
Claude building the targeting, no Sales Nav needed.

**What does it cost?**
Sales Nav (which you already have) plus Evaboot credits — a lead with a verified
email is 2 credits, roughly $0.04–0.07 each depending on your plan. The Nous
dedup keeps you from spending credits on companies you already have.
