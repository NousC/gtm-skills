---
name: lead-builder
description: Describe who you want, give example companies, or even name one person ("the founder of acme.com"), and it builds an enriched lead list — the right decision maker at each matching company and a verified email — saved straight into a Nous lead list. Claude turns your prompt into a precise, multi-variant targeting spec (industries, keywords, titles, size), runs it through Apollo's people search, reveals and verifies the emails, and dedupes against Nous before you spend. It runs in three phases: sharpen the spec, confirm the ICP and a rough cost, then run. No Sales Navigator needed.
---

# Lead-list builder

## What it does

You describe who you want — *"founders of outbound GTM agencies, 1 to 10 people,
US"* — and it builds the list: the **decision maker** at each matching company
and a **verified email**, deduped against what you already have and saved into a
**Nous lead list** ready for outbound.

The intelligence is up front: **Claude turns your prompt into a sharp targeting
spec** — the industries, keyword variants, titles, size bands, and exclusions
that find exactly your ICP — then runs it through Apollo's people search. You can
seed it three ways and it reverse-engineers the rest:

- a **description** — *"cold email agencies, under 10 people"*
- **example companies** — *"like anna-agency.com and bravo-collective.com"*
- a **single person** — *"the founder of acme.com, find me more like them"*

The flow: **Claude (targeting) → Apollo (people) → Nous (dedup) → FullEnrich
(email) → MillionVerifier → Nous (save).** No Sales Navigator, no lookalike
subscription — Claude is the lookalike engine.

## How to invoke

`/lead-builder` — or *"find me 5,000 founders of outbound GTM agencies, 1 to 10
employees, US, like anna-agency.com."*

## First-run setup (you, the agent, run this once as a short interview)

Detect what's connected, ask only for what's missing, one thing at a time.

**1. Apollo — the people + email (required).** Check for `APOLLO_API_KEY`.
Missing → "I find the decision maker at each company through Apollo. The people
search costs no per-call credits, but Apollo's API is **only on a paid plan**
(from ~$59/mo; full programmatic access sits on the higher tiers), so that
subscription is the real floor — it is not free. You only pay *extra* to reveal
emails. Use a **master API key**. `export APOLLO_API_KEY=...` (apollo.io →
Settings → API)."

**2. Nous — dedup + where the list lands (required).** Check `NOUS_API_KEY`.
Missing → "`export NOUS_API_KEY=pk_xxx` (opennous.cloud → Settings → API keys).
It dedupes against your pipeline and saves the list. Free at opennous.cloud."

**3. Optional, for a higher match rate and clean sends.**
`FULLENRICH_API_KEY` (a charge-on-found waterfall across 20+ sources for the
emails Apollo misses) and `MILLIONVERIFIER_API_KEY` (a ~$0.0005/email verify
pass for catch-alls). Both are pure pay-as-you-go, no monthly floor.

---

## Phase 1 — Build the targeting spec (this is the whole game)

A vague brief wastes credits and finds noise. Your job is to turn whatever the
user gives you into a **precise, structured targeting spec** before anything
runs. This is where Claude earns its place — you are the lookalike engine.

**Read the seed, whichever form it takes:**

- **A description** → expand the *category* into every synonym a real company in
  that space would use. "Outbound agencies" is not one keyword — it's
  `lead generation, demand generation, appointment setting, cold email, outbound,
  SDR, RevOps, GTM engineering, pipeline generation, fractional GTM`. Generate
  the full set; a single keyword finds a fraction of the market.
- **Example companies** → look at what those companies actually do (their site,
  their positioning) and extract the shared profile: industry, the keywords in
  their description, size band, tech. Build the spec from the pattern, not from
  the names.
- **A single person** ("the founder of acme.com") → reverse-engineer them. What
  is their title and seniority, what does their company do, how big is it, what
  category is it in? That profile becomes the search — this is the "find people
  like this one" path, done by reasoning, not a vendor.

**Produce a structured spec:**

```
Titles:      Founder, Co-founder, CEO, Owner, Managing Partner
Seniority:   owner, founder, c_suite
Industries:  Marketing & Advertising
Keywords:    lead generation, demand generation, appointment setting,
             cold email, outbound, SDR, RevOps, GTM
Size:        1–10 employees
Geo:         United States
Exclude:     staffing, recruiting, PR
Target:      5,000
```

Ask only for what's genuinely missing (size, geo, buyer title, count) — one
question at a time, conversationally. Infer everything you can.

### Cross-check against the saved ICP

Before you confirm, pull the workspace's own GTM profile and compare it to what
they asked for — the saved ICP is the source of truth, the one-off brief might
be a new segment or a slip:

```bash
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

- **Aligned** → say so in one line ("matches your saved ICP") and move on.
- **Diverges** → surface it plainly and ask before spending:
  > "Your saved ICP is **insurance companies, 200+ employees** — but you asked
  > for **agency founders, 1–10**. Target this one-off, your saved ICP, or a
  > refined mix? And should I update the saved ICP?"

## Phase 2 — Confirm the spec and a rough cost, then wait

Show the spec back, preview the real volume with an Apollo count call, and lay
out the rough cost. **Wait for an explicit yes** before revealing a single
email.

```
Targeting I'll run:
  Titles:    Founder / Co-founder / CEO / Owner
  Niche:     outbound GTM agencies (10 keyword variants)
  Size:      1–10 employees
  Geo:       United States
  Target:    5,000

Apollo preview: ~4,800 people match. After dedup against your Nous pipeline,
~3,500 net-new. Expected ~2,800 verified emails.

Rough cost: people search adds no per-call cost (paid Apollo plan is the
floor); ~$140 email reveal + ~$1 verify  ≈  $141 in usage.
Run it?
```

Estimate: a paid Apollo plan (~$59/mo+) is the floor that unlocks the API; the
people search then adds no per-call cost; email ≈ net-new founders × match-rate
× ~$0.05 (FullEnrich, charge-on-found); verify is rounding error.

## Phase 3 — Run, save to Nous, report back

On a yes, run the pipeline, create the list, and hand the user back — they don't
wait in the terminal:

> "Started. About 2,800 leads will land in your new Nous list
> **'Founder · outbound GTM agencies · 1–10 · US'** over the next few minutes.
> I'll write each one in as its email verifies."

**Default list title:** `<Buyer> · <niche> · <size> · <geo>`.

---

## The pipeline (the real calls)

### 1. Find the people — Apollo people search (no per-call credits; paid plan required)

Run the spec as a people search. This consumes **no per-call credits** (though
Apollo's API needs a paid plan) — it returns
obfuscated identity (`id`, `first_name`, `last_name_obfuscated`, `title`, org
name, `has_email`). Run **several keyword variants** and union the results — one
query never covers the whole category.

```bash
curl -s -X POST "https://api.apollo.io/api/v1/mixed_people/api_search" \
  -H "x-api-key: $APOLLO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "person_titles": ["Founder","Co-Founder","CEO","Owner"],
        "person_seniorities": ["owner","founder","c_suite"],
        "q_organization_keyword_tags": ["lead generation","demand generation","appointment setting","cold email","outbound"],
        "organization_num_employees_ranges": ["1,10"],
        "organization_locations": ["United States"],
        "per_page": 100, "page": 1 }'
```

Take one decision maker per company (the `id` + the org `primary_domain`).
Seeding from example companies? First pull *their* profile
(`mixed_companies/search` on the seed domains) to read the industry/keywords,
then feed those into the people search above.

### 2. Dedup by domain — Nous (free, before you spend)

```bash
curl -s -X POST "https://api.opennous.cloud/v2/dedup" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "domains": ["acme.com","beta.io","gamma.co"] }'
```

Keep only `status === 'net_new'`. You never pay to reveal an email for a company
you already have.

### 3. Reveal the email — FullEnrich waterfall (charge-on-found)

For each net-new person, reveal the email through the waterfall — it
cross-sources 20+ providers and returns a deliverability status, which is more
trustworthy than any single source.

```bash
# Submit (async)
curl -s -X POST "https://app.fullenrich.com/api/v2/contact/enrich/bulk" \
  -H "Authorization: Bearer $FULLENRICH_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name":"lead-builder", "data":[
        { "first_name":"Jane","last_name":"Doe","domain":"acme.com",
          "enrich_fields":["contact.work_emails"] } ] }'
# → { "enrichment_id": "<uuid>" }  then poll:
curl -s "https://app.fullenrich.com/api/v2/contact/enrich/bulk/<uuid>" \
  -H "Authorization: Bearer $FULLENRICH_API_KEY"
```

Returns `most_probable_work_email { email, status }` (DELIVERABLE /
HIGH_PROBABILITY / CATCH_ALL). Keep DELIVERABLE and HIGH_PROBABILITY. (If you
only have an Apollo key, `people/bulk_match` reveals emails too — cheaper, but
single-source, so verify harder.)

### 4. Verify the catch-alls — MillionVerifier (~$0.0005/email)

Send CATCH_ALL / uncertain emails to MillionVerifier; keep only `ok`/`good`.
DELIVERABLE emails skip this step.

### 5. Save to a Nous lead list

```bash
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name": "Founder · outbound GTM agencies · 1–10 · US", "source": "lead_builder" }'

curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "leads": [
        { "name":"Jane Doe", "email":"jane@acme.com", "company":"Acme",
          "fields": { "title":"Founder", "niche":"outbound GTM agency",
                      "source":"Lead-list builder",
                      "matched_on":"keywords", "enriched_by":"fullenrich" } } ] }'
```

Insert as they verify so the list fills in live. Then point the user at
`campaign-writer`.

## Hard rules — never break these

- **Build the spec first.** Never run a vague brief — Phase 1 is the product. A
  rich, multi-variant query is what makes the list targeted and reliable.
- **Cost and approval before any reveal.** Phase 2 shows the estimate and waits.
- **Dedup before you reveal.** Run `/v2/dedup`; reveal only `net_new`.
- **Never guess an email.** Unverifiable contacts are flagged, not invented.
- **Name the list from the ICP** and tell the user where to find it.
- **Stamp the lead source.** Highly recommended: set `fields.source` to
  `"Lead-list builder"` on every lead, so reply rates can be compared across lead
  sources later (this skill vs Sales Nav vs inbound vs LinkedIn engagers).

## Customize / Set up

- **Seed three ways** — a description, example domains, or one person to match.
- **Tune the spec** — titles, seniority, size, geo, keyword breadth, exclusions.
- **Match rate vs cost** — Apollo's own reveal is cheapest; add FullEnrich to
  lift the rate and trust, MillionVerifier to protect deliverability.
- **Rename the default list** if you want.

## Frequently asked questions

**Can it find people similar to one person or company?**
Yes — that's the seed-from-a-person path. Give it "the founder of acme.com" and
it reads that person's title, seniority, and what their company does, then builds
a people search for the same profile. Claude does the similarity reasoning, so
you don't need a lookalike vendor.

**Why no Sales Navigator or DiscoLike here?**
This is the no-Sales-Nav path — its one platform subscription is Apollo (not
Sales Nav), and Claude's targeting replaces a similarity vendor like DiscoLike.
If you have Sales Navigator and want the sharpest possible targeting, use
`sales-nav-builder` instead.

**What does a run cost?**
A paid Apollo plan (from ~$59/mo) is the floor — it unlocks the API and the
people search, which then adds no per-call cost. On top of that you pay only to
reveal emails — ~$0.05 per found email, charge-on-found. So ~$0.05 per verified
lead in usage, on top of the Apollo subscription. The Nous dedup
keeps you from paying that reveal twice. Phase 2 always shows the estimate first.

**Do I have to wait in the terminal?**
No. Phase 3 starts the run and hands you back — the leads stream into the Nous
list as they verify.
