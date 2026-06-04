---
name: linkedin-engagers
description: From one or more LinkedIn creators' profile URLs, scrape the commenters AND reactors on their recent posts via Apify HarvestAPI, dedupe them, and score every one against your Nous ICP (icp / score / reason, keeping both fit and non-fit, labelled). It then finds verified emails on the ICP-qualified ones by their LinkedIn URL — so you still get the email even when the scraped company is wrong — and saves the ICP-segmented list into both Nous and a Google Sheet, with every engagement recorded as a public signal. Use to turn a creator's audience into a high-intent, ICP-scored outbound list, or refresh it on a schedule.
---

# High-intent LinkedIn scraper

## What it does

You give it one or more LinkedIn creators' profile URLs. It scrapes the people
who **commented** and **reacted** on their posts from the last **two weeks**,
dedupes them against your pipeline, scores each one against your **Nous ICP**,
and saves the clean, high-fit list into **both Nous (as a lead list) and a
Google Sheet** — with every engagement recorded on each person as a public
signal.

These are warm leads by definition: they showed up for the content. Pointed at
the right creators every two weeks, this is a steady source of in-market,
on-ICP prospects with the source of the signal traced on every record.

The flow: **Apify → Claude (ICP score) → Prospeo (email on keepers) → Nous → Google Sheet.**

## How to invoke

`/linkedin-engagers` — or *"create a lead list of everyone who engaged with
these creators' recent posts: linkedin.com/in/anna-mraz, linkedin.com/in/tom-becker."*

The skill will ask for what it needs:

- **LinkedIn profile URLs** of the creators (`https://www.linkedin.com/in/<slug>/`).
- **Window in days** (default `14`, two weeks; widen to `21` for three weeks).
- **List name** (default `<creator-slug> · <YYYY-WW>`).
- **Google Sheet** to mirror into (a `GOOGLE_SHEET_ID`, optional).
- At the end: **outbound tool** to push to (HeyReach / Smartlead / none).

## First-run setup (you, the agent, run this once as a short interview)

Detect what's already set, ask only for what's missing, one thing at a time.

**1. Apify — the scrape.** Check for `APIFY_TOKEN`. Missing → "I scrape the
engagers through Apify, about $2.50 per 300 leads. Add your token once:
`export APIFY_TOKEN=apify_api_xxx` (apify.com → Settings → Integrations)."

**2. Nous — the lead list + ICP scoring.** Check for `NOUS_API_KEY`. Missing →
"Add your Nous key: `export NOUS_API_KEY=pk_xxx` (opennous.cloud → Settings →
API keys). No account yet? It's free at opennous.cloud." ICP scoring also needs
a **GTM profile** set (opennous.cloud → GTM Context); without one the skill
keeps every engager and skips the fit step — tell them, then continue.

**3. Email enrichment (optional but recommended).** Check for `PROSPEO_API_KEY`
(LinkedIn URL → verified email + clean company, one synchronous call) or
`FULLENRICH_API_KEY` (charge-on-found waterfall). Missing → "I find the email
from each person's LinkedIn URL, which also fixes the company name. Add
`export PROSPEO_API_KEY=...` (prospeo.io) to get emails; without it the list
saves leads-only."

**4. Optional outputs.** Only if they want them: mirror to a sheet
(`export GOOGLE_SHEET_ID=1AbC...`, plus `gcloud auth login` once) or push to
outbound (`export HEYREACH_API_KEY=...` for LinkedIn, `SMARTLEAD_API_KEY` for
email).

Then confirm the creators and the window before spending.

The skill calls **three** Apify HarvestAPI actors — your token must have access:

| Actor (slash form) | Used for |
|---|---|
| `harvestapi/linkedin-post-search`    | recent posts by a creator |
| `harvestapi/linkedin-post-comments`  | commenters on a post |
| `harvestapi/linkedin-post-reactions` | reactors on a post |

In Apify URLs the slash becomes `~` (e.g. `harvestapi~linkedin-post-search`).

## Hard caps — these protect your Apify spend

These are not options. They exist because we have burned credits learning them.

| Cap | Value | Why |
|---|---|---|
| Window | **14 days default, 21 max** | Window where a creator's post is still warm. |
| Post age floor | **48 hours** | Younger posts haven't accumulated engagement yet — mining early misses 80%+. |
| Posts mined per run | **8** | Each mined post = 2 paid Apify runs. Cap keeps worst case ≈ $3.40. |
| Engagers per post | **100** | Viral posts can return thousands; we cap. |
| Post-search limit | **30** | Max posts fetched before age/state filtering. |

Worst-case backfill per creator = 1 search + 8 × 2 = **17 paid Apify runs**.

## Core philosophy

**Engagement is the signal, not the form-fill.** Comments and reactions in
public beat form fills. Treat that as the lead.

**Score against the ICP, keep both, spend only on keepers.** Score every engager
against your GTM profile (icp / score / reason). Keep the non-ICP ones too,
labelled — never silently drop them — and find emails only on the ICP-qualified,
so the variable cost lands on the people you'd actually contact.

**The LinkedIn URL is the key, not the company.** The scraped company is
unreliable, so don't derive a domain from it. Feed the profile URL straight to a
LinkedIn-native finder, which returns the email and the real company at once.

**Trace every record back to why it's there.** Every person gets a
`public_signal` on their record in Nous — the post URL, type, timestamp. Six
months later you still know why they're in the list.

**Dedupe like your job depends on it.** Normalize the LinkedIn URL before
matching (casing, trailing slash, `www.`, query string). The bulk-add endpoint
also dedupes server-side; treat that as a safety net, not the strategy.

**Never re-mine a post.** Keep local state of mined post URLs. Re-running the
skill the next day must not re-pay Apify for posts already mined.

## The process

Claude runs these steps in order. Refuse to proceed if a guard fails — never
"try anyway".

### 1. Validate inputs (refuse on garbage — protects Apify spend)

```js
// Profile URL guard
/^https?:\/\/(www\.)?linkedin\.com\/(in|company)\//.test(profileUrl)
// Post URL guard (applied at step 3)
/^https?:\/\/(www\.)?linkedin\.com\/.+activity-/.test(postUrl)
```

If a URL fails the regex, **do not call Apify**. Report the bad URL and stop.

### 2. Load local state — `~/.claude/skills/linkedin-engagers/state.json`

```json
{ "minedPosts": { "<normalized_post_url>": "<ISO_timestamp>" } }
```

Create the file if missing. Posts in `minedPosts` will be skipped at step 4.
The file survives across runs — that is the whole point.

### 3. Fetch recent posts (one paid run per creator)

```bash
curl -s -X POST \
  "https://api.apify.com/v2/acts/harvestapi~linkedin-post-search/run-sync-get-dataset-items?token=$APIFY_TOKEN&timeout=120" \
  -H "Content-Type: application/json" \
  -d '{ "profileUrls": ["<CREATOR_URL>"], "maxItems": 30, "sortBy": "date" }'
```

`sortBy: "date"` is required. `"recent"` is rejected by the actor.

### 4. Resolve each post's timestamp, then filter

HarvestAPI rotates the `postedAt` shape — handle all four:

| Shape | How to read |
|---|---|
| `postedAt: "2026-05-10T..."` (ISO string) | `Date.parse(p.postedAt)` |
| `postedAt: "3w"` (relative) | parse the suffix |
| `postedAt: { timestamp, date, postedAgoShort }` | `obj.timestamp ?? Date.parse(obj.date)` |
| `postedAtTimestamp: <ms>` (old) | use directly |

If none resolve, **decode the activity ID from the URL** as a last resort.
Posts whose timestamp can't be determined are **dropped, not included**.

Then filter (`WINDOW_DAYS` defaults to 14):

```
postedAt < now - 48h             → drop (engagement not settled)
postedAt < now - WINDOW_DAYS     → drop (out of window)
normalized URL ∈ minedPosts      → drop (already mined)
remaining                        → cap at 8 (MAX_MINES_PER_RUN)
```

### 5. For each surviving post, fetch comments and reactions

Two paid runs per post. Run them in parallel where possible.

```bash
# Commenters — param is `posts`, NOT `postUrls`
curl -s -X POST \
  "https://api.apify.com/v2/acts/harvestapi~linkedin-post-comments/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "posts": ["<POST_URL>"], "maxItems": 100 }'

# Reactors
curl -s -X POST \
  "https://api.apify.com/v2/acts/harvestapi~linkedin-post-reactions/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "posts": ["<POST_URL>"], "maxItems": 100 }'
```

### 6. Normalize the response — profile is under `actor`

Every engager row has a nested `actor` object. Flatten it:

```js
{
  linkedin_url: row.actor.linkedinUrl ?? row.profileUrl ?? row.linkedinUrl,
  name:        row.actor.name        ?? row.fullName,
  title:       row.actor.position    ?? row.actor.headline ?? row.position ?? row.headline,
  company:     row.actor.company     ?? row.company,
  source_type: <"comment" | "reaction">,   // from WHICH actor returned the row
  comment_text: row.commentText,            // only for comments
  reaction_type: row.reactionType,          // only for reactions
}
```

`source_type` is determined by **which actor returned the row**, not a field
on the row. There is no flat `engagementType` field.

### 7. Dedupe within this run, normalizing URLs

```js
function normalizeLinkedInUrl(u) {
  return u.toLowerCase()
          .replace(/^https?:\/\//, "https://")
          .replace(/\/\/www\./, "//")
          .replace(/[?#].*$/, "")
          .replace(/\/+$/, "");
}
```

Group rows by `normalizeLinkedInUrl(linkedin_url)`. If the same person both
commented and reacted, **comment wins** (stronger signal). Keep both
engagements in a `signals: []` array for step 11.

### 8. Score each engager against your ICP

Read your **full** GTM profile from Nous, then score every deduped engager
against it. Same ICP model `sales-nav-builder` uses; it runs in the skill on
your Claude tokens — Nous holds the ICP, the skill reasons with it.

```bash
# Your GTM profile / ICP — same data as the get_gtm_profile MCP tool
curl -s "https://api.opennous.cloud/v2/workspace/facts?categories=ICP,Market,Product,Pricing,Competitors" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

For each engager, write three values into its `fields` and **keep it regardless**:

- `icp`: `true | false` — does this person match the ICP?
- `icp_score`: `0–100` — how strong the fit is.
- `icp_reason`: one short sentence — why, citing what matched or missed.

Judge on what you actually have: the **headline/title**, any **company named in
the headline**, and the **engagement signal itself** — they showed up for this
creator's topic, which is its own ICP cue. **Don't drop the non-ICP ones** — they
stay in the list, labelled, leads-only (no email spend). Only `icp: true`
engagers go on to email enrichment (step 8.5), so the variable cost lands only on
the people you'd actually contact.

If no GTM profile is set, mark `icp: null` and `icp_reason: "no ICP set"`, keep
everyone, and tell the operator to fill in GTM Context. Never invent a score.

### 8.5. Find emails on the ICP-qualified — by LinkedIn URL (this fixes the email gap)

You have a reliable **profile URL** but the scraped `company` is usually wrong,
so deriving a domain to find an email fails. **Skip the company guess entirely:**
feed the LinkedIn URL straight to a LinkedIn-native finder, which returns the
verified work email **and** the real company + domain. Do this **only for
`icp: true` net-new engagers** — never pay to enrich non-ICP or duplicates.

```bash
# Prospeo — LinkedIn URL → verified work email (synchronous, one call)
curl -s -X POST "https://api.prospeo.io/linkedin-email-finder" \
  -H "X-KEY: $PROSPEO_API_KEY" -H "Content-Type: application/json" \
  -d '{ "url": "<engager_linkedin_url>" }'
# Alternative: FullEnrich waterfall — accepts linkedin_url, charge-on-found,
# cross-sources 20+ providers for a higher hit rate.
```

Returns the work `email` plus the resolved `company`, `domain`, and `position`.
Write the verified email and clean company back onto the lead (overwriting the
unreliable scraped company). No key set → skip enrichment, save leads-only, and
tell the operator they can add `PROSPEO_API_KEY` to get emails.

### 9. Create the Nous lead list

The create response returns both the `id` and the `workspace_id` — capture both.

```bash
curl -s -X POST "https://api.opennous.cloud/api/lead-lists" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name": "<LIST_NAME>", "source": "linkedin_engagers" }'
# → { "lead_list": { "id": "<LIST_ID>", "workspace_id": "<WS>", ... } }

# Declare the icp columns so the list filters ICP vs non-ICP in the Nous UI.
# This PATCH REQUIRES workspaceId in the body — use the <WS> above.
curl -s -X PATCH "https://api.opennous.cloud/api/lead-lists/<LIST_ID>" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "workspaceId": "<WS>", "columns": [
        {"key":"title","label":"Title"}, {"key":"company","label":"Company"},
        {"key":"icp","label":"ICP"}, {"key":"icp_score","label":"ICP score"},
        {"key":"icp_reason","label":"Why"}, {"key":"creator","label":"From creator"} ] }'
```

### 10. Bulk-add ALL net-new engagers, ICP-tagged (keep both)

Add **every** net-new engager — ICP and non-ICP. ICP-qualified ones carry their
verified email; non-ICP ones are saved leads-only. Use `name` + a `fields` JSONB
(the verified API shape — top-level `full_name`/`icp_fit` are ignored):

```bash
curl -s -X POST "https://api.opennous.cloud/api/lead-lists/<LIST_ID>/leads" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "importDuplicates": false, "leads": [
      { "name":"Jane Doe", "email":"jane@acme.com",
        "linkedin_url":"<normalized>", "company":"Acme",
        "fields": { "title":"Founder", "icp":true, "icp_score":84,
                    "icp_reason":"GTM founder engaging on outbound — core ICP",
                    "creator":"<creator_url>", "enriched_by":"prospeo" } },
      { "name":"Bob Lurker", "linkedin_url":"<normalized>",
        "fields": { "title":"Designer", "icp":false, "icp_score":18,
                    "icp_reason":"design role, outside ICP", "creator":"<creator_url>" } } ] }'
```

`importDuplicates: false` is the default — only set `true` after warning the
operator. Response: `{ inserted, skipped }`. **Leads land immediately but Nous
resolves them in the background, so names and the `icp` tags fill in ~10s after
insert** — tell the operator the list populates shortly, don't report it empty.

**If steps 9 or 10 fail (non-2xx), do not abort the run.** Warn the operator,
then continue to steps 11-13 so the engagements are still recorded as public
signals and written to the Google Sheet. The lead list is recoverable; the
scraped engagers are not worth re-paying Apify for.

### 11. Record every engagement as a public signal

Do this for **every engager — including the ones skipped as duplicates in step
10**. The engagement is interesting even if the person already exists in Nous.

```bash
curl -s -X POST "https://api.opennous.cloud/v2/observations" \
  -H "Authorization: Bearer $NOUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "focus": "<engager_linkedin_url>",
    "observations": [{
      "kind": "event",
      "property": "interaction.linkedin_engagement_comment",
      "value": {
        "post_url": "<POST_URL>",
        "creator_url": "<CREATOR_URL>",
        "comment_text": "<text>"
      },
      "source": "apify_linkedin",
      "method": "api",
      "observed_at": "<ISO>",
      "external_id": "linkedin_engagers:<POST_URL>:<engager_url>"
    }]
  }'
```

- `interaction.linkedin_engagement_comment` for comments.
- `interaction.linkedin_engagement_like` for reactions (use the reaction_type
  in `value` if present).
- `external_id` is what makes reruns safe — Nous skips a duplicate observation
  on the same `(source, external_id)`.

### 12. Mirror the list into a Google Sheet (optional)

If `GOOGLE_SHEET_ID` is set, append the same high-fit leads as rows so the team
has them in a sheet too. Append with the Sheets API using a short-lived token
from `gcloud auth print-access-token` (the operator must have run
`gcloud auth login` once):

```bash
curl -s -X POST \
  "https://sheets.googleapis.com/v4/spreadsheets/$GOOGLE_SHEET_ID/values/Leads!A1:append?valueInputOption=USER_ENTERED" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{ "values": [
        ["<name>", "<linkedin_url>", "<title>", "<company>", "<icp_fit>", "<creator_url>", "<observed_at>"]
      ] }'
```

If a Google Sheets MCP connector is attached instead, use it. If neither is
available, write a local `engagers-<list>.csv` and tell the operator where it
is. Never block the run on the sheet step.

### 13. Update state — mark posts as mined

After all observations are written for a post, add its normalized URL to
`state.json` with `now()`. **Only update state after the post's engagers are
successfully recorded** — never mark a post mined when the call failed
halfway through.

### 14. Push to an outbound tool (optional)

Ask the operator:

> *"Push this list to an outbound tool? HeyReach, Smartlead, or none?"*

- **HeyReach** (LinkedIn outreach): use the HeyReach public API to add the
  leads to a campaign or list. Ask for the campaign/list id; key in
  `HEYREACH_API_KEY`.
- **Smartlead** (email): `POST https://server.smartlead.ai/api/v1/campaigns/{campaignId}/leads?api_key=$SMARTLEAD_API_KEY`. Ask for `campaignId`.
- **Other / none**: stop. The list is in Nous and the sheet; the operator
  handles export.

For tools without an env var set, ask the operator to set it and re-run that
step only — never abort the whole run.

To reach out over **both LinkedIn and email**, add an email-find step before
this one (an enrichment provider or the Nous record's email) and push to
HeyReach for LinkedIn and Smartlead for email.

### 15. Summarise

```
List "<name>" created in Nous and mirrored to the sheet.
Creators: <C>. Mined: <P> posts (skipped <S> already-mined, <Y> too young, <O> out of window)
Apify spend: <C> + <P>*2 paid runs
Engagers: <total>, deduped to <unique>
ICP: <fit> fit, <maybe> maybe, <off> filtered out
Leads inserted: <inserted>, skipped (already existed): <skipped>
Engagements recorded as public signals: <obs_count>
Pushed to <outbound>: <pushed> (skipped <push_skipped>)
```

## Hard rules — never break these

- **Refuse on bad URL.** No Apify call without the profile/post URL guard passing.
- **48h floor.** Never mine a post younger than 48 hours.
- **8 posts per run.** Never raise `MAX_MINES_PER_RUN` without checking the
  Apify quota first.
- **100 engagers per post.** Pass `maxItems: 100` on every comments/reactions call.
- **Dedupe with normalized URLs.** Lowercase, strip trailing slash, strip
  `www.`, strip query/fragment — before comparing.
- **Score honestly, keep both.** Judge ICP fit only on the fields you have, write
  `icp` / `icp_score` / `icp_reason`, and keep the non-ICP ones labelled — never
  silently drop them. Mark `icp: null` when there's no GTM profile.
- **Emails on ICP keepers only, by LinkedIn URL.** Never derive an email from the
  unreliable scraped company; use the profile URL. Never enrich non-ICP or dupes.
- **State after success, never before.** A failed mid-flight call must leave
  the post NOT marked mined, so the next run retries it.
- **Observations for everyone.** Even leads skipped as duplicates get their
  engagement recorded.
- **`source: "apify_linkedin"`, `method: "api"`** on every observation.

## Run on a schedule

To mine a creator every two weeks automatically:

```bash
# Cron — every 14 days at 9am
0 9 */14 * *  cd ~/work && claude -p "/linkedin-engagers https://www.linkedin.com/in/adamrobinson808/"
```

The skill is idempotent — the state file plus the `external_id` pattern make
reruns safe.

## Frequently asked questions

**What's the worst-case Apify spend for one creator?**
17 paid runs (1 post-search + 8 × 2 engager pulls), roughly $3.40, or about
$2.50 per 300 leads on HarvestAPI pricing at time of writing.

**How does the ICP scoring work?**
The skill reads your full GTM profile from Nous and scores each engager — `icp`
(true/false), `icp_score` (0–100), `icp_reason` — on their headline, any company
in it, and the engagement signal itself. It **keeps both** the ICP and non-ICP
ones, labelled, so the list filters ICP vs non-ICP in Nous. Set up your GTM
profile at opennous.cloud → GTM Context first; without one, scoring is skipped
and everyone is kept.

**How do I get the email when the scraped company is wrong?**
That's exactly the problem this fixes. The scraped company is unreliable, so the
skill doesn't try to derive a domain from it. It sends the person's **LinkedIn
profile URL** — which you do have reliably — to a LinkedIn-native finder
(Prospeo, or a FullEnrich waterfall), which returns the verified work email
together with the real company and domain. It only does this for the
ICP-qualified people, so you don't pay to enrich the off-ICP ones.

**Why save into both Nous and a Google Sheet?**
Nous is the resolved record the rest of your stack reads from. The sheet is for
the humans who want to eyeball or hand the list to a teammate. Both get the
same ICP-scored leads.

**Will it create duplicates of people I already have in Nous?**
No. The bulk-upload endpoint matches on email and normalized `linkedin_url`
and skips existing records. Their engagement is still recorded as a public
signal on their existing record.

**Will it re-mine posts on subsequent runs?**
No. The local `state.json` tracks mined post URLs; mined posts are skipped at
step 4. Delete the entry if you ever want to force a re-mine.
