---
name: company-people
description: Give it a list of companies (LinkedIn company URLs or domains) and it finds the decision maker at each one — founder / co-founder / CEO / owner / GTM lead — with a verified email, saved into a Nous lead list. It scrapes each company's LinkedIn People tab via Apify (HarvestAPI), filters to the right titles, pulls the email, verifies it with NeverBounce inside the skill, flags catch-all/risky, and writes the keepers to Nous. This is the MOST RELIABLE people layer (LinkedIn has nearly every founder), so it's the catch-all that fills the gaps the databases miss — and it works standalone on any company list.
---

# Company → decision-maker (LinkedIn people layer)

## What it does

You give it **companies** (LinkedIn company URLs, or domains) and it returns the
**decision maker at each** with a **verified email**, saved into a **Nous lead
list**. LinkedIn is the source of truth — nearly every agency founder is on it —
so this is the **most reliable way to get the person**, and the natural catch-all
for companies the data vendors (`lookalike-builder` / `lead-builder`) couldn't
find a person for.

```
companies (LinkedIn URLs)  →  Apify HarvestAPI scrape the /people/ tab
                           →  filter Founder/Co-Founder/CEO/Owner/GTM/Head of Growth
                           →  email from the scrape
                           →  NeverBounce verify (inside this skill)
                           →  route: valid keep · catch-all flag · invalid drop
                           →  Nous lead list
```

## How to invoke

`/company-people` — paste or point it at a list of company LinkedIn URLs (or
domains), or hand it the gap companies from `lookalike-builder`.

## First-run setup (run once as a short interview)

**1. Apify (required) — the scrape.** Check `APIFY_TOKEN`. The actor is
`harvestapi/linkedin-company-employees` ("No Cookies", returns name, title,
LinkedIn URL, and an email). **It needs a one-time permission approval** in the
Apify console the first time (full-access actor) — if a run returns
`full-permission-actor-not-approved`, send the user the approval URL it gives.

**2. NeverBounce (required) — the verify.** Check `NEVERBOUNCE_API_KEY`. The skill
verifies every scraped email itself (does NOT rely on Nous's built-in Verify), so
the list lands already graded.

**3. Nous (required) — where the list lands.** Check `NOUS_API_KEY` (`pk_`) +
`workspaceId`. It dedupes and stores the list.

**4. Optional — Prospeo (`PROSPEO_API_KEY`)** for the fallback layer (find a fresh
email off the LinkedIn URL when NeverBounce says the scraped email is *invalid*).
Off by default; see the fallback note in Phase 4.

---

## Phase 1 — The company list + the title filter

Take the companies. Preferred input is **LinkedIn company URLs**
(`https://www.linkedin.com/company/<slug>`) — `lookalike-builder` already returns
one per company (`link.linkedin`). If you only have domains, resolve each to its
LinkedIn company URL first (the AI-Ark company record carries it; else a quick
lookup).

Default **decision-maker titles** (the `jobTitles` filter, applied at scrape time):

```
Founder, Co-Founder, CEO, Owner, Managing Partner, Managing Director,
GTM Engineer, Head of Growth
```

Keep it tight — for a 3–20 person agency the People tab is small, so this filter
returns essentially just the founder/leadership. Tune only if the user names a
different buyer.

## Phase 2 — Preview the cost, then WAIT (the gate)

This skill spends real money (Apify compute + NeverBounce). **Always show the
estimate and wait for an explicit yes before scraping.**

```
N companies → ~N decision-makers (≈1 founder each on a small agency).
Cost: scrape ≈ $0.012/profile (HarvestAPI Full+email) + verify ≈ $0.004/email
      → roughly $<N × 0.016> for N companies.
Proceed?
```

Never scrape before the user confirms.

## Phase 3 — Scrape the people (Apify HarvestAPI)

Batch the company URLs (the actor accepts many in `companies`). Run sync and read
the dataset:

```bash
curl -s --max-time 600 -X POST \
  "https://api.apify.com/v2/acts/harvestapi~linkedin-company-employees/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "companies": ["https://www.linkedin.com/company/zevenue", "..."],
        "jobTitles": ["Founder","Co-Founder","CEO","Owner","Managing Partner","GTM Engineer","Head of Growth"],
        "maxItems": 500,
        "profileScraperMode": "Full + email search ($12 per 1k)" }'
```

Output is an array; per person read: `firstName` + `lastName`, `linkedinUrl`,
`headline` / `currentPosition` (title), `companyWebsites`, and **`emails[0].email`**
(+ the scrape's own `status` / `catchAllDomain` — a hint, not the verdict). NOTE:
the JSON can contain raw control characters — parse leniently (`strict=False`).
If a run returns the permission error, surface the approval URL and stop.

## Phase 4 — Verify each email with NeverBounce (inside the skill), then route

Do NOT trust the scrape's self-graded status, and do NOT defer to Nous Verify —
verify here so the list lands graded. Call NeverBounce single-check per email:

```bash
curl -s "https://api.neverbounce.com/v4/single/check?key=$NEVERBOUNCE_API_KEY&email=jane@acme.com&address_info=1"
# → { "status":"success", "result":"valid|invalid|disposable|catchall|unknown", "flags":[...] }
```

Route by `result`:
- **`valid`** → keep, `email_status = verified`.
- **`catchall`** → keep, `email_status = risky` (a catch-all domain can't be
  confirmed by ANY tool — this is the domain's nature, not a bad address; send at
  your discretion).
- **`unknown`** → keep, `email_status = risky`.
- **`invalid` / `disposable`** → drop the email. *(Fallback, optional/off by
  default: if `PROSPEO_API_KEY` is set, re-find off the `linkedinUrl` via Prospeo,
  then re-verify — this is the ONLY case Prospeo helps, never for catch-all.)*

## Phase 5 — Save to a Nous lead list

Create the list, then insert the keepers with their verified status. Same pattern
as the other builders.

```bash
# create
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name":"LinkedIn · agency founders", "source":"company_people" }'
# → { "lead_list": { "id":"<LIST_ID>", "workspace_id":"<WS>" } }

# insert (one per decision-maker)
curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "source":"Company-people (LinkedIn)", "importDuplicates": true, "leads":[
        { "name":"Yusuf Ahmed", "email":"yusuf@zevenue.com", "company":"Zevenue",
          "linkedin_url":"https://www.linkedin.com/in/itsyusufahmed",
          "fields": { "title":"Founder", "domain":"zevenue.com",
                      "email_status":"verified", "source":"company_people" } } ] }'
```

Map: `name` ← firstName+lastName, `linkedin_url` ← `linkedinUrl`, `company` ← the
input company, `email` ← the verified address, `fields.email_status` ← the
NeverBounce verdict, `fields.title` ← headline/position. Then dedup is handled by
Nous; point the user at `signal-scan` / `content-scan` next.

## Hard rules — never break these

- **Confirm the cost before scraping.** Phase 2 shows the estimate and waits.
- **Verify inside the skill (NeverBounce).** Don't rely on the scrape's status or
  Nous's built-in Verify — grade every email here so the list lands clean.
- **Catch-all = keep + flag, never "fix" with Prospeo.** No tool verifies a
  catch-all domain; Prospeo helps only on `invalid`.
- **Never invent an email.** Drop `invalid`/`disposable`; flag `risky`.
- **Stamp the lead source** (`company_people`) so reply rates compare across sources.

## How this fits the bigger picture

The find-skills split by SOURCE, each its strength:
- `lookalike-builder` (AI-Ark) → discovers the **lookalike companies** (irreplaceable).
- `lead-builder` (Apollo) / `sales-nav-builder` (Sales Nav) → database people search.
- **`company-people` (this) → the LinkedIn people layer: the most reliable way to
  get the founder at a known company, and the catch-all that fills the gaps the
  databases miss.** Runs standalone on any company list, or takes the
  AI-Ark-blank companies from `lookalike-builder` for maximum coverage.

## Cost

Apify HarvestAPI: **Short $4/1k profiles** (person only) or **Full+email $12/1k**
(person + email — used here so emails are bundled, cheaper than scraping +
Prospeo). NeverBounce verify ≈ **$0.003–0.008/email**. So ≈ **$0.016 per
decision-maker**, all in. Prospeo fallback (optional) ≈ $0.02 only on the
`invalid` slice.

## FAQ

**Why LinkedIn instead of a database?** LinkedIn has nearly every founder; the
databases (AI-Ark ~50% on tiny agencies, Apollo varies) miss many. This is the
catch-all that gets the people they don't have.

**Do I pay for emails separately?** No — the scrape's Full+email mode bundles the
email, and NeverBounce just verifies it (cheap). You only reach for Prospeo on the
small `invalid` slice, if you enable the fallback.

**Why not Nous's built-in Verify?** The skill verifies itself (NeverBounce) so the
list arrives already graded, in one flow — no second manual step.
