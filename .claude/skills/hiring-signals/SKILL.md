---
name: hiring-signals
description: Discover net-new, in-market accounts from hiring data. A job posting is a timed buying trigger, so this finds companies that just posted the role you sell into (via TheirStack), scores them against your Nous ICP, drops anyone already in your pipeline, records the hiring signal as context on each account in Nous, saves a scored lead list, and drafts a short cold-email sequence that opens on the signal. The trigger-to-outreach skill.
---

# Hiring-signal prospecting

## What it does

A job posting is a buying trigger with a timestamp. A company hiring three SDRs
has a fresh, urgent need for what you sell. This skill **discovers** those
companies from hiring data (you don't bring a list), **scores** them against
your ICP, **drops** anyone you're already working, **records** the hiring signal
as context in Nous, **saves** a scored lead list, and **drafts** a short cold
sequence that opens on the signal. Trigger to outreach in one run.

The flow: **TheirStack → Claude Code → Nous.**

## How to invoke

`/hiring-signals` — or *"find 25 companies hiring SDRs in the last two weeks
that fit my ICP."*

## First-run setup (you, the agent, run this once as a short interview)

Detect what's connected, ask only for what's missing, one thing at a time.

**1. TheirStack — the hiring data.** Check for `THEIRSTACK_API_KEY`. Missing →
"I find the hiring companies through TheirStack. Get a key at
app.theirstack.com/settings/api, then `export THEIRSTACK_API_KEY=...`. It charges
**1 credit per job returned**, so I'll always preview the count and confirm
before spending."

**2. Nous — ICP scoring, dedupe, and the record.** Check whether the Nous MCP is
connected (try `get_gtm_profile`); also set `NOUS_API_KEY` for the lead-list
write.
- Not connected → "Connect Nous in one line:
  `claude mcp add nous -e NOUS_API_KEY=<your-key> -- npx -y @opennous/mcp`. Key
  at opennous.cloud → Settings → API keys (the `pk_` one). Free at
  opennous.cloud."

**3. GTM profile set?** The scoring and the email angle both read it. If
`get_gtm_profile` is empty: "Set up your GTM profile at opennous.cloud → GTM
Context first, so I score and write from what you actually sell."

**4. Apollo — to find the decision maker.** Check for `APOLLO_API_KEY`. Missing →
"To turn each company into an emailable lead, I find the buyer through Apollo.
Add your key once: `export APOLLO_API_KEY=...` (apollo.io → Settings → API). It
charges a credit per email revealed, so I'll preview and confirm before
spending. Prefer Prospeo? Say so and I'll use that instead." Without an
enrichment key the skill stops at companies + signals and creates no leads.

**5. Optional** — a sequencer key (Instantly / Smartlead / HeyReach) to push the
finished list. Skip for a draft-only run.

## Core philosophy

**Hiring is the trigger, timing is the edge.** A fresh, relevant posting beats a
static list. Lead with the signal while it's warm.

**Net-new and on-ICP only.** Score against your profile and dedupe against your
pipeline before anything reaches a list. No noise.

**The signal becomes context.** Record *why* each account is a lead (the role,
the count, when posted) on its Nous record, so every later touch can reference
it and the Mind can learn from it.

**Spend deliberately.** Credits cost money. Preview, confirm, then reveal.

## The process

### 1. Define the trigger

From the prompt plus your GTM profile's ICP firmographics, build the search:
the role(s) you sell into, the recency, and the company shape (size, country,
optionally tech stack).

### 2. Discover the companies (TheirStack)

```bash
curl -s -X POST "https://api.theirstack.com/v1/jobs/search" \
  -H "Authorization: Bearer $THEIRSTACK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "job_title_or": ["SDR", "BDR", "Sales Development Representative"],
    "posted_at_max_age_days": 14,
    "job_country_code_or": ["US"],
    "min_employee_count": 20,
    "max_employee_count": 500,
    "limit": 25,
    "page": 0
  }'
```

Useful filters: `job_title_or` / `job_title_pattern_or`, `posted_at_max_age_days`
(or `posted_at_gte`/`posted_at_lte`), `job_country_code_or`,
`min_employee_count` / `max_employee_count`, `industry_or`,
`company_technology_slug_or` (companies using a given tech).

**Cost — state it before you spend.** TheirStack charges **1 credit per job
returned**. With a tight role filter that's about 1 job per company, so a
25-company run is roughly **25 credits**. Before revealing, say it plainly and
wait for a yes — e.g. *"25 companies match, about 25 credits at 1 per job.
Reveal them?"* Never reveal without showing the number first. Start with a small
`limit`; widen only on confirmation.

### 3. Collapse to companies + score the signal

One row per company: the role(s) they're hiring, how many, and when posted.
That's the signal you'll lead with — "hiring 2 SDRs, posted 3 days ago".

**Rate the signal's strength 1-10** (recency × relevance). A fresh post (under
~2 weeks) for a role your product directly serves is high-intent (**8-10**); an
older post or a loosely-related role is lower (**3-5**). The strength rides with
the lead and tells the sequence how hard to lean on the trigger — strong signals
open pain-led, weaker ones fall back to the segment's common pain.

### 4. Score against your ICP (Nous)

Read your GTM profile (`get_gtm_profile`) and judge each company on what you
have (industry, size, the role they're hiring). Drop the clearly off-ICP ones;
keep `fit` and `maybe`. Never invent a score.

### 5. Dedupe against your pipeline (Nous)

For each surviving company, check whether it's already in your workspace (by
domain or name). Drop the ones you're already working. Keep only **net-new**.

### 6. Resolve the decision maker (Apollo)

For each net-new, on-ICP company, find the **buyer for the trigger** — the
person who'd own the purchase, not the recruiter. Map the hiring role to the
buyer (cross it with what you sell, from the GTM profile):

- hiring SDRs / BDRs → VP / Head / Director of Sales (or RevOps)
- hiring marketers → VP / Head of Marketing
- hiring engineers → CTO / VP Engineering

Find them, then reveal the email:

```bash
# Find the buyer at the company
curl -s -X POST "https://api.apollo.io/v1/mixed_people/search" \
  -H "X-Api-Key: $APOLLO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "organization_domains": ["acme.com"],
        "person_titles": ["VP Sales","Head of Sales","Director of Sales"],
        "per_page": 1 }'

# Reveal the email (1 Apollo credit)
curl -s -X POST "https://api.apollo.io/v1/people/match" \
  -H "X-Api-Key: $APOLLO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "first_name": "Jane", "last_name": "Doe", "domain": "acme.com" }'
```

**Confirm before revealing** — like the TheirStack step: *"25 buyers to reveal,
about 25 Apollo credits. Go?"* Capture name, email, linkedin_url, and title. If
no confident buyer or email resolves for a company, hold it as a company +
signal account rather than guessing a contact. (Prospeo works the same way if
that's the key you set.)

### 7. Save the decision maker as a lead, with the hiring data in its columns

The decision maker is a **lead** — not a contact, not a company — so it goes in
the leads table. The leads table already supports custom columns, so **no schema
change is needed**: the list's `columns` define them, each lead's `fields` JSONB
holds the values.

- Create or reuse a **"Hiring signals" lead list** (`POST /api/lead-lists`), and
  set its columns once (`PATCH /api/lead-lists/:id` with `columns`):
  `hiring_role`, `hiring_date`, `hiring_count`, `job_url`, `icp_score`,
  `signal_strength`, `signal_source`, `source`.
- Insert the decision maker (`POST /api/lead-lists/:id/leads`) — identity in the
  normal columns, the hiring context in `fields`:

```json
{ "name": "Jane Doe", "email": "jane@acme.com",
  "linkedin_url": "https://www.linkedin.com/in/janedoe", "company": "Acme",
  "fields": { "hiring_role": "SDR", "hiring_date": "2026-05-30",
              "hiring_count": 2, "job_url": "https://...", "icp_score": 82,
              "signal_strength": 9, "signal_source": "theirstack",
              "source": "Hiring signals" } }
```

Workspace-wide dedup on email / linkedin_url runs automatically. Also record the
hiring signal on the lead's timeline (`signal.hiring` observation) for durable
context, but the campaign-ready data lives in the lead's columns.

### 8. Export to a campaign, or draft the sequence

The lead list is now export-ready. Push it to your sequencer (Instantly /
Smartlead / HeyReach) with the hiring `fields` as merge variables, hand it to
`campaign-writer` for the full data-driven campaign, or draft a short
signal-grounded sequence here, three emails with per-lead variables:

```
E1  Noticed {{company}} is hiring {{count}} {{role}}s. Scaling that team usually
    means {the pain you solve}. {one-line value}. Worth a quick look?
E2  A new angle or proof, still tied to the hire.
E3  A short breakup.
```

Variables (`{{first_name}}`, `{{company}}`, `{{role}}`, `{{count}}`,
`{{signal}}`) fill per lead, so it's one sequence that personalizes at scale.
Save it as a draft in Nous, push it to a sequencer if a key is set, or **hand
the list to the `campaign-writer` skill** for the full data-driven campaign
(learns from what's replied, suppresses, records the copy per variant).

## Hard rules — never break these

- **Confirm before spending.** Always preview the job count and get a yes before
  revealing (1 credit per job).
- **Net-new and on-ICP only.** Score and dedupe before anything is saved.
- **Record the signal.** A company isn't done until the hiring trigger is on its
  Nous record — that's the whole point.
- **Stamp the lead source.** Highly recommended: set `fields.source` to
  `"Hiring signals"` on every lead (distinct from `signal_source`, the provider),
  so reply rates can be compared across lead sources later.
- **Lead with the trigger.** Every sequence opens on the hire; never a generic
  cold open.

## Customize / Set up

- **Tune the trigger** — roles, recency, company size, country, and tech stack
  (`company_technology_slug_or`, e.g. companies on a competitor's tool).
- **Cap the spend** — set a max job count per run.
- **Resolve buyers** — add an enrichment key to turn accounts into contacts.
- **Choose the outreach** — draft only, push to your sequencer, or hand to
  `campaign-writer` for the full campaign.
- **Schedule it weekly** — a fresh batch of in-market accounts every Monday.

## Frequently asked questions

**How is this different from a lead list I buy?**
A bought list is static. This is timed: every company on it just signaled a need
this week, scored to your ICP, with the reason recorded so your outreach can
reference it.

**What does a run cost?**
TheirStack charges 1 credit per job returned. With a tight `job_title_or` that's
about 1 job per company, so a 25-company run is roughly 25 credits. The skill
states the estimate and confirms before spending — you always see the number
first.

**Do I get contacts or just companies?**
Companies + the signal by default. Add an enrichment key and it resolves the
buyer persona per company so the sequence has a real recipient.
