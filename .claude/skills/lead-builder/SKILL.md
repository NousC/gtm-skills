---
name: lead-builder
description: Describe your ICP or give example companies to match, and it builds an enriched lead list — lookalike companies, the decision maker at each, and a verified email — saved straight into a Nous lead list. It runs in three phases: it sharpens your prompt, shows the ICP and a rough cost for approval, then runs the pipeline and tells you the leads are landing in Nous. Discovery via DiscoLike, people + email via Apollo (with a FullEnrich waterfall), deduped against Nous before you spend.
---

# Lead-list builder

## What it does

You describe who you want — *"agency founders, 1 to 10 people, US, like these
three"* — and it builds the list: **lookalike companies**, the **decision maker**
at each, and a **verified email**, deduped against what you already have and
saved into a **Nous lead list** ready for outbound. It works in three guided
phases so you never run a vague brief or spend a credit you didn't approve.

The flow: **DiscoLike → Nous (dedup) → Apollo → FullEnrich → Nous (save).**

## How to invoke

`/lead-builder` — or *"find me 5,000 agency founders, 1 to 10 employees, US,
similar to anna-agency.com and bravo-collective.com."*

## First-run setup (you, the agent, run this once as a short interview)

Detect what's connected, ask only for what's missing, one thing at a time.

**1. DiscoLike — lookalike company discovery.** Check for `DISCOLIKE_API_KEY`.
Missing → "I find the similar companies through DiscoLike. Get a key at
app.discolike.com → account → keys, then `export DISCOLIKE_API_KEY=...`."
DiscoLike is a subscription that converts to credits, not pure pay-as-you-go —
it starts at **$99/mo (the Starter plan), and that $99 becomes your credit
balance**. Discovery then draws those credits down at **$3.50 per 1,000 new
company records** (cheaper on higher tiers), and discovered records are cached
in your account for 90 days, so re-running the same search costs nothing.

**2. Apollo — the decision maker + email.** Check for `APOLLO_API_KEY`. Missing →
"I find the founder at each company and reveal the email through Apollo (use a
master API key). `export APOLLO_API_KEY=...` (apollo.io → Settings → API)."

**3. Nous — dedup + where the list lands.** Check `NOUS_API_KEY`. Missing →
"`export NOUS_API_KEY=pk_xxx` (opennous.cloud → Settings → API keys). It dedupes
against your pipeline and saves the list. Free at opennous.cloud."

**4. Optional, for a higher match rate.** `FULLENRICH_API_KEY` (a charge-on-found
waterfall across 20+ sources for the emails Apollo misses) and
`MILLIONVERIFIER_API_KEY` (a ~$0.0005/email verify pass for catch-alls).

---

## Phase 1 — Shape the prompt (never run a vague brief)

A vague ICP wastes credits. Before anything, sharpen it. If the brief is thin
("agency founders, 1 to 10 people"), say so and ask the questions that make the
lookalike good — one at a time, conversationally:

- **What kind of agency / company?** The niche is everything — "paid-social
  agencies" finds a sharp set; "agencies" finds noise.
- **Example companies to match?** Ask for 1–10 company domains. This is
  DiscoLike's lookalike input and the single biggest quality lever — "like X"
  beats any filter.
- **Geography?** Country (and state if relevant).
- **Which buyer?** The title to target — Founder, CEO, Owner.
- **How many leads?**

Only move on when you have a clear niche **or** example domains, plus size,
geography, buyer title, and a target count.

### Cross-check against the saved ICP

Before you confirm, pull the workspace's own GTM profile from Nous and compare
it to what they just asked for — the saved ICP is the source of truth, the
one-off brief might be a new segment or a slip:

```bash
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

- **Aligned** → say so in one line ("matches your saved ICP") and move on.
- **Diverges** → surface it plainly and ask before spending:
  > "Your saved ICP is **insurance companies, 200+ employees** — but you asked
  > for **agency founders, 1–10**. Should I target this one-off, your saved ICP,
  > or a refined mix? And do you want me to update the saved ICP?"

Never run a list that contradicts the saved ICP without an explicit yes — it's
the cheapest way to catch a wrong-segment run before it costs credits.

## Phase 2 — Confirm the ICP and a rough cost, then wait

Translate to the structured ICP, preview the real volume with DiscoLike's
low-cost `/count`, and lay out the rough cost. **Wait for an explicit yes**
before spending.

```
ICP I'll search:
  Lookalikes of: anna-agency.com, bravo-collective.com
  Niche:    paid-social agencies
  Size:     1–10 employees
  Geo:      United States
  Buyer:    Founder / CEO / Owner
  Target:   5,000 companies

DiscoLike preview: ~5,000 match. After dedup against your Nous pipeline,
~3,500 net-new. Expected ~2,800 verified founder emails.

Rough cost: ~$18 discovery + ~$140 enrichment + ~$1 verify  ≈  $159.
Run it?
```

Estimate: DiscoLike record cost ≈ companies × $3.50/1k (Starter rate, less on
higher tiers); Apollo/FullEnrich ≈ net-new founders × match-rate × ~$0.05;
verify is rounding error. DiscoLike bills per company it returns, so the Nous
dedup mainly saves the **enrichment** spend (you only reveal emails for net-new),
and the 90-day record cache means a repeated search is free.

## Phase 3 — Run, save to Nous, report back

On a yes, kick off the pipeline (below), create the list, and tell the user it's
running — they don't wait in the terminal:

> "Started. About 2,800 leads will land in your new Nous list
> **'Founder · paid-social agencies · 1–10 · US'** over the next few minutes.
> I'll write each one in as its email verifies."

**Default list title:** `<Buyer> · <niche> · <size> · <geo>` — e.g.
`Founder · paid-social agencies · 1–10 · US`.

---

## The pipeline (the real calls)

### 1. Discover lookalike companies — DiscoLike

```bash
# Preview the count first (low-cost)
curl -s "https://api.discolike.com/v1/count?domain=anna-agency.com&domain=bravo-collective.com&country=US&employee_range=1,10" \
  -H "x-discolike-key: $DISCOLIKE_API_KEY"

# Then discover
curl -s "https://api.discolike.com/v1/discover?domain=anna-agency.com&domain=bravo-collective.com&country=US&employee_range=1,10&max_records=5000&min_similarity=60" \
  -H "x-discolike-key: $DISCOLIKE_API_KEY"
```

Pass 1–10 seed `domain`s for the lookalike (or `icp_text=...` for a pure
description). Returns companies with `domain`, `name`, `similarity` (0–100),
`employees`, `address`, `description`. Keep the company `domain`s.

### 2. Dedup by domain — Nous (free, before you spend)

```bash
curl -s -X POST "https://api.opennous.cloud/v2/dedup" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "domains": ["acme.com","beta.io","gamma.co"] }'
```

Response: `{ results: [{ kind:'domain', value, status }], summary: { net_new, known, ... } }`.
Keep only `status === 'net_new'`. You never pay to enrich the companies you
already have.

### 3. Find the decision maker — Apollo people search (free, obfuscated)

Search people at the net-new domains (up to 1,000 domains per call) filtered to
the buyer titles. This consumes **no credits** — it returns obfuscated identity
(`id`, `first_name`, `last_name_obfuscated`, `title`, org name, `has_email`).

```bash
curl -s -X POST "https://api.apollo.io/api/v1/mixed_people/api_search" \
  -H "x-api-key: $APOLLO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "person_titles": ["Founder","CEO","Owner"],
        "q_organization_domains_list": ["acme.com","beta.io"],
        "per_page": 100, "page": 1 }'
```

Take one decision maker per company (the `id`).

### 4. Reveal email + LinkedIn — Apollo bulk match (paid, metered)

```bash
curl -s -X POST "https://api.apollo.io/api/v1/people/bulk_match" \
  -H "x-api-key: $APOLLO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "details": [ {"id":"<apollo_person_id>"}, ... ],
        "reveal_personal_emails": false }'
```

Returns `email`, `email_status`, `linkedin_url`, org `primary_domain`, and
`credits_consumed` (meter your spend). For anyone Apollo can't find an email
for, fall to the waterfall.

### 5. Waterfall the misses — FullEnrich (optional, charge-on-found)

```bash
# Submit (async)
curl -s -X POST "https://app.fullenrich.com/api/v2/contact/enrich/bulk" \
  -H "Authorization: Bearer $FULLENRICH_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name":"lead-builder", "data":[
        { "first_name":"Jane","last_name":"Doe","domain":"acme.com",
          "linkedin_url":"https://www.linkedin.com/in/janedoe",
          "enrich_fields":["contact.work_emails"] } ] }'
# → { "enrichment_id": "<uuid>" }  then poll:
curl -s "https://app.fullenrich.com/api/v2/contact/enrich/bulk/<uuid>" \
  -H "Authorization: Bearer $FULLENRICH_API_KEY"
```

Returns `most_probable_work_email { email, status }` (DELIVERABLE / HIGH_PROBABILITY
/ CATCH_ALL). Keep DELIVERABLE and HIGH_PROBABILITY.

### 6. Verify the catch-alls — MillionVerifier (optional, ~$0.0005/email)

Send the CATCH_ALL / uncertain emails to MillionVerifier's API; keep only the
ones it returns `ok`/`good`. DELIVERABLE emails skip this step.

### 7. Save to a Nous lead list

```bash
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name": "Founder · paid-social agencies · 1–10 · US", "source": "lead_builder" }'

curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "leads": [
        { "name":"Jane Doe", "email":"jane@acme.com",
          "linkedin_url":"https://www.linkedin.com/in/janedoe", "company":"Acme",
          "fields": { "title":"Founder", "niche":"paid-social agency",
                      "similarity":74, "enriched_by":"apollo" } } ] }'
```

Insert as they enrich so the list fills in live. Then point the user at
`campaign-writer` to write to them.

## Hard rules — never break these

- **Shape the prompt first.** Never run on a vague brief — Phase 1 is not
  optional.
- **Cost and approval before any spend.** Phase 2 shows the estimate and waits
  for a yes.
- **Dedup before enrich.** Run `/v2/dedup` on the domains; enrich only `net_new`.
- **Never guess an email.** Unverifiable contacts are flagged, not invented.
- **Name the list from the ICP** and tell the user where to find it.

## Customize / Set up

- **Set the target count, geography, and buyer titles.**
- **Lookalike vs description** — give example domains for the sharpest match, or
  `icp_text` for a broad ICP.
- **Match rate vs cost** — Apollo-only is cheapest; add FullEnrich to lift the
  email rate, MillionVerifier to protect deliverability.
- **Rename the default list** if you want.

## Frequently asked questions

**Why DiscoLike and not just Apollo for the lookalike?**
Apollo can only match firmographics. DiscoLike matches what a company actually
does, from its website — so "similar to X" finds companies in the same business,
not just the same size band.

**What does a run cost?**
Two parts. **Discovery** is DiscoLike — a subscription that converts to credits,
from **$99/mo (Starter)**, drawn down at **$3.50 per 1,000 new companies** (less
on higher tiers), with records cached 90 days so repeats are free. **Enrichment**
is the email reveal, ~$0.05 per found email, charge-on-found. The Nous dedup
keeps you from paying that reveal twice on companies you already have. Phase 2
always shows the estimate first.

**Do I have to wait in the terminal?**
No. Phase 3 starts the run and hands you back — the leads stream into the Nous
list as they verify.
