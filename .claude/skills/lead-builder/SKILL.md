---
name: lead-builder
description: Describe your ICP in plain English, or give a few examples to match, and it builds an enriched lead list — company, decision maker, email, and LinkedIn — via waterfall enrichment across Apollo, Prospeo, and any provider you've keyed in. It dedupes against your pipeline and saves the list straight into Nous, ready for outbound. Before it spends a credit, it shows your parsed ICP and the exact cost and waits for your go.
---

# Lead-list builder

## What it does

Tell it who you want to reach — *"100 agency founders with 1 to 10 employees who
focus on paid social, similar to these three"* — and it builds the list for you:
the company, the **decision maker**, their **email** and **LinkedIn**, enriched
through a **waterfall** so the match rate is high. It dedupes against what you
already have and saves the list straight into a **Nous lead list**, ready to drop
into outbound. Before it spends anything, it shows you the **ICP it parsed** and
the **exact cost**, and waits for your yes.

The flow: **Apollo + Prospeo → Claude Code → Nous.**

## How to invoke

`/lead-builder` — or *"find me 100 agency founders, 1 to 10 employees, who focus
on paid social, similar to {example 1}, {example 2}."*

## First-run setup (you, the agent, run this once as a short interview)

Detect what's connected, ask only for what's missing, one thing at a time.

**1. Apollo — discovery + first-pass enrichment.** Check for `APOLLO_API_KEY`.
Missing → "I find the people and their emails through Apollo. Add your key once:
`export APOLLO_API_KEY=...` (apollo.io → Settings → API). It charges a credit per
email revealed, so I'll show the estimate and confirm before spending."

**2. Waterfall providers (optional but recommended).** A second email-finder so
the ones Apollo misses still get an email: `export PROSPEO_API_KEY=...`
(prospeo.io), or whichever finder you use (Findymail, Dropcontact). The skill
only pays the next source when the first comes up empty.

**3. Nous — where the list lands + dedupe.** Check `NOUS_API_KEY` (and the MCP if
you want it connected). Missing → "Add your Nous key: `export NOUS_API_KEY=pk_xxx`
(opennous.cloud → Settings → API keys). The list saves here and dedupes against
your pipeline. Free at opennous.cloud."

## Core philosophy

**Show the ICP and the cost before you spend.** No surprise credit burn. The
operator sees exactly who they're targeting and what it costs, and says go.

**Waterfall to a high match rate.** Try the cheapest/best source first, fall to
the next only when it misses. A lead with no email is a lead you can't use.

**Net-new only.** Dedupe against the Nous pipeline before saving, so you never
pay to rediscover someone you already have.

**The list is the start of outbound.** Enrich it fully and save it where the
campaign writer can pick it up.

## The process

### 1. Parse the ICP

Turn the description (and any examples) into structured search criteria:
titles, company size, industry/keywords, and geography. If they gave **examples
to match**, extract the shared pattern from those companies — size, industry,
the keywords in how they describe themselves — and search for that lookalike.

### 2. Show the ICP and the cost, then wait

Before any spend, lay it out plainly and confirm:

```
Here's the ICP I'll search:
  Titles:    Founder, CEO, Owner
  Size:      1–10 employees
  Focus:     paid-social marketing agencies
  Geography: United States
Target: 100 leads. Estimated cost: up to ~100 enrichment credits
(Apollo first; Prospeo only on the misses). Want me to run it?
```

Never search-and-reveal before a yes.

### 3. Discover (Apollo)

```bash
curl -s -X POST "https://api.apollo.io/v1/mixed_people/search" \
  -H "X-Api-Key: $APOLLO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "person_titles": ["Founder","CEO","Owner"],
        "organization_num_employees_ranges": ["1,10"],
        "q_organization_keyword_tags": ["marketing agency","paid social"],
        "person_locations": ["United States"],
        "per_page": 25, "page": 1 }'
```

Paginate (`page`) until you reach the target count. Each result is a person +
their company + their LinkedIn URL (the email is masked here — reveal it next).

### 4. Waterfall-enrich each lead

For every lead, get a usable email by trying sources in order, stopping at the
first hit:

```bash
# 1) Apollo reveal
curl -s -X POST "https://api.apollo.io/v1/people/match" \
  -H "X-Api-Key: $APOLLO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "first_name":"Jane","last_name":"Doe","domain":"acme.com" }'

# 2) On a miss, your next finder (e.g. Prospeo) by name + domain
#    POST https://api.prospeo.io/email-finder  (X-KEY: $PROSPEO_API_KEY)
```

Capture `email`, `linkedin_url`, `title`, `company`, `domain`. A lead that no
source can verify is flagged `email: unverified` and kept aside, not guessed.

### 5. Dedupe against your pipeline (Nous)

Drop anyone already in the workspace (by email or normalized linkedin_url). Keep
only **net-new** — the bulk-insert dedupes again as a safety net.

### 6. Save to a Nous lead list

Create or reuse a lead list and insert the enriched leads:

```bash
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name": "Agency founders — paid social", "source": "lead_builder" }'

curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "leads": [
        { "name":"Jane Doe", "email":"jane@acme.com",
          "linkedin_url":"https://www.linkedin.com/in/janedoe",
          "company":"Acme",
          "fields": { "title":"Founder", "icp_match":"paid social",
                      "enriched_by":"apollo" } } ] }'
```

The list is now in Nous, deduped and enriched — hand it to `campaign-writer`, or
export it to your sequencer.

## Hard rules — never break these

- **ICP + cost before the spend.** Always show the parsed ICP and the credit
  estimate and get a yes before discovering or revealing.
- **Waterfall in order, stop on a hit.** Never pay two sources for the same
  email.
- **Net-new only.** Dedupe against Nous before saving.
- **Never fabricate an email.** Unverifiable leads are flagged, not guessed.

## Customize / Set up

- **Set the target count** and the geography.
- **Order the waterfall** — which finder runs first, which is the fallback.
- **Tune the match** — lean on the examples for a lookalike, or on the
  description for a broad ICP.
- **Name the list** and choose whether to hand it to `campaign-writer` or export
  it to your sequencer.

## Frequently asked questions

**Description or examples — which works better?**
Both. A description gives a broad, precise ICP; examples find lookalikes of
companies you already know convert. Give both and it uses the examples to
sharpen the description.

**What does a run cost?**
Roughly one enrichment credit per lead, less when the first source hits and the
waterfall doesn't fire. The skill shows the estimate and confirms before
spending — you always see the number first.

**What about leads with no findable email?**
They're flagged `unverified` and set aside, never guessed. You can still keep
them for LinkedIn-only outreach.
