# sales-nav-builder — Refinement Spec

**Source:** live run on 2026-06-04 building "outbound / GTM agency founders" list.
**Audience:** the developer refining the `sales-nav-builder` skill.
**TL;DR:** The skill over-indexes on keyword targeting. Real precision comes from
*structural* Sales Nav filters + *downstream* filtering/scoring in Nous. And the
money model should split into two paid stages with our filtering in the middle.

---

## Part A — What we learned the hard way (empirical findings from the run)

1. **Sales Nav's keyword box is fragile with Boolean `NOT`.**
   - Long `(...) AND NOT (...)` clauses → **0 results**, every time.
   - Even short `(positives) NOT (negatives)` → **0 results** when the leading `(`
     is dropped on paste, or on smart-quote conversion, or past a length limit.
   - A **flat positive `OR`** list (no parens, no NOT) works reliably.
   - **Conclusion: the skill must stop instructing users to build exclusions in the
     keyword box.** Use a flat positive OR for *inclusion*, and do *all* exclusion
     downstream. (See Part B.)

2. **The real ICP discriminators are STRUCTURAL, not keyword.**
   Legacy local agencies and modern AI-native GTM agencies *both* say "lead
   generation" / "demand generation" — keywords can't separate them. What worked:
   - **Years at current company ≤ 5** — single biggest lever. Instantly removes the
     16–24-year-tenure legacy marketing/SEO/design veterans.
   - **Industry choice** — see #3.
   - **Headcount 1–10** — matches the saved Nous ICP.
   The skill currently leads with "expand the category into keyword variants." It
   should lead with these structural filters and treat keywords as secondary.

3. **"Advertising Services" industry is a legacy-agency trap.**
   It's where Word-of-Mouth-Marketing / SEO / web-design / branding shops live.
   Modern AI-native GTM agency founders (the actual ICP — e.g. Zeveue, Growth Today,
   Workflows.io) are tagged under **Marketing Services / Business Consulting &
   Services / IT & Services / Software Development / Technology, Information &
   Internet**. The skill's example recipe should be updated.

4. **Sales Nav hard-caps any export at 2,500 leads.**
   A search returning 3,000 silently truncates on extraction. The skill should warn
   the user and push them to tighten *below 2,500* so the export is complete.

5. **Modern-vocabulary keywords are the precision lever, not generic ones.**
   `Clay`, `Claude`, `AI-native`, `GTM`, `AI SDR`, `RevOps`, `go-to-market`,
   `outbound`, `cold email` — terms a 20-year word-of-mouth shop never uses.
   Generic `lead generation` / `demand generation` pull in everyone; keep them only
   when structural filters are already doing the separating.

---

## Part B — The product change the user wants (the important part)

### B1. Split into two PAID stages, with our filtering in the middle

Today the skill exports "with emails" in one Evaboot pass, then filters — so you
**pay to find+verify emails on leads you throw away** (junk company types + dupes).
The user wants the spend split cleanly:

```
Stage 1  EXTRACT (pay)      Evaboot "No Emails" export → LinkedIn data only.
                            Cost: 1 credit / lead. This is the LinkedIn-scrape we pay for.
Stage 2  FILTER (free)      Inside Nous: drop excluded company types + dedup + ICP-score.
Stage 3  EMAILS (pay)       Evaboot Email Finder API — ONLY on the surviving ICP-qualified,
                            net-new leads. Cost: 1 credit / email actually found.
Stage 4  SAVE              Lead list, ICP-segmented (see B2).
```

The principle the user stated: *"we only look up the people on LinkedIn, so we pay
for that, then we filter them, then we only pay for the filtered ones back in
Evaboot."* Extraction credits are unavoidable (data must come out), but **email
credits — the variable cost — are spent only on leads we're keeping.**

**Optimal order inside Stage 2 (do all three before any email spend):**
1. Company-type exclusion filter (deterministic, by company name/description).
2. Nous dedup by domain (drop leads already in pipeline — no point paying for them).
3. ICP scoring (Part B2) — only ICP-qualified leads proceed to Stage 3 emails.

### B2. ICP scoring + segmentation INSIDE the lead list (don't discard non-ICP)

The user does **not** want non-ICP leads deleted — they want them **kept and
labelled** so the list is filterable:

- Run an **ICP score** on every extracted lead (against the saved Nous ICP/Market
  facts — already queryable via `/v2/workspace/facts?categories=ICP,Market`).
- Tag each lead `icp: true | false` (or a 0–100 score + threshold) on its record.
- The lead list can then be **filtered "ICP vs non-ICP"** — you see the qualified
  core *and* the rest you also pulled, without losing visibility into the non-ICP
  leads (useful for refining the ICP definition over time).
- **Only the ICP-qualified leads get emails found** (Stage 3). Non-ICP leads stay
  in the list leads-only (no email spend), flagged, for visibility / future use.

This turns the lead list from a flat dump into a segmented asset, and makes the ICP
definition itself something you can inspect and tighten against real pulled data.

---

## Part C — Concrete skill changes, phase by phase

**Phase 1 (build the search):**
- Replace the keyword-first guidance with a **structural-filters-first** recipe:
  Years-at-company ≤ 5 · headcount band · the *right* industries (explicitly call
  out "NOT Advertising Services for modern agencies") · seniority/title.
- Keywords = a **flat positive `OR`** of modern vocabulary only. Explicitly tell the
  user **not** to put exclusions in the keyword box (it breaks).
- Add the **2,500 export cap** warning: tighten below it for a complete pull.

**Phase 2 (confirm + cost):**
- Show the **two-stage** cost: Stage-1 extraction credits now, Stage-3 email credits
  estimated *after* filtering/dedup/scoring. Make clear emails are only charged on
  survivors. Use the real credit model in Part E.

**Phase 3 (extract):**
- Default the Evaboot extension export to **"No Emails"** (leads-only), NOT
  "With Emails". This is the credit-saver and should be the documented default.

**New Phase 3.5 (filter + score — the new middle):**
- Deterministic company-type exclusion (list below).
- Nous dedup by domain.
- ICP scoring → tag `icp: true/false`.

**Phase 4 (emails + save):**
- Evaboot Email Finder API on ICP-qualified net-new only.
- Save to lead list with the ICP segmentation flag on every record.

**Exclusion list (company name/description) — make this a documented, editable default:**
> design · web design · web development · website · graphic · UX/UI · SEO ·
> search engine · branding · logo · creative · recruiting · staffing ·
> recruitment · talent · headhunting · executive search · word of mouth ·
> PR / public relations · social media management · digital marketing

Match on company *identity*, not stray profile mentions, so real GTM agencies that
merely say "marketing" once aren't nuked.

---

## Part D — Real Evaboot pricing (pulled from evaboot.com/pricing, 2026-06-04)

| Action | Cost |
|---|---|
| 1 lead exported (data only) | **1 credit** |
| 1 email found + verified | **1 credit** (charged only when an email is found) |
| 1 email verified (already have it) | 0.5 credit |
| 1 lead with verified email (combined) | **2 credits** |

Plans: $9/mo = 100 credits → 500 / 1.5k / 4k / 8k / 20k / 50k / 100k / 200k.
Effective ~$0.09/credit at entry tier, ~$0.04–0.05 on bigger plans. Email finder
hit rate 60–80%. API endpoints: Sales Navigator (export trigger), Enrichment,
Email Finder, Email Verifier. Confirmed email-finder endpoint:
`POST https://api.evaboot.com/v1/email-finder/`.

**Worked example (this run, ~1,300 leads):**
- Stage 1 export 1,300 leads-only → ~1,300 credits
- Stage 2 filter (~20% junk) + dedup → ~850 ICP net-new (free)
- Stage 3 emails on 850 × ~70% hit → ~600 credits
- **Total ≈ 1,900 credits** vs ~2,200 for naive "With Emails" on all 1,300 — and
  cleaner, because no email spend on junk/dupes.

---

## Part E — Open questions for the dev — RESOLVED on the 2026-06-05 live run

1. **Headless export: YES.** `POST /v1/extractions/url/` takes the Sales Nav search
   URL + `enrich_email` (`"none"|"matching"|"all"`, default `"none"` = leads-only),
   returns a `search_id`; poll `GET /v1/extractions/` until `EXECUTED`, then pull all
   leads from `GET /v1/extractions/<id>/`. No CSV handoff. The Chrome extension is
   only needed **once** to connect the Sales Nav session (`/v1/quota/` →
   `has_valid_salesnav`). `POST /v1/search-builder/` even turns NL → a Sales Nav URL.
2. **ICP scoring runs in the skill (LLM), not a Nous endpoint.** Confirmed working:
   pull `get_gtm_profile`, score in the skill, write `icp` / `icp_score` /
   `icp_reason` into each lead's `fields`; they survive Nous resolution and filter
   in the list UI.
3. **Misses are free: confirmed** by the pricing/quota model (1 credit per email
   *found*). Extraction itself is **1 credit/lead, deducted on completion** (1,500 →
   295 after a 1,205 pull) — not free, and not mid-run.

---

## Part F — What the 2026-06-05 live run changed (the big correction)

Building a real ~1,205-lead "GTM agency founders" list surfaced one structural bug
in the v1 skill and several smaller ones. SKILL.md has been rewritten to fix them.

1. **DO NOT pre-filter leads out of the list — this was the worst bug.** v1 excluded
   "wrong company types" before saving, so on the real run 1,114 of 1,205 *paid*
   leads were dropped. The operator was (rightly) furious: you paid to extract them,
   so import them. New model: **import every extracted lead, score every lead, let
   the ICP score be the filter inside Nous. Delete is manual-only.**
2. **Evaboot's `Matches Filters: NO` is NOT off-ICP.** It mostly means "didn't
   contain the literal keyword string." On this run, ~87 genuine ICP leads (RevOps /
   outbound / GTM founders) were sitting in the `NO` pile. Keep the flag as a
   *signal/field*, never a gate.
3. **The blocklist must be word-boundary on company identity, and only LOWER the
   score — never drop.** A naive substring scan of the whole bio matched `ui` inside
   "building" and `design` inside "designed", nuking 77 of 91 real leads. Use
   `\b<term>\b` on company name + industry.
4. **Credit model clarified.** Extraction = 1 credit/lead, deducted on completion.
   Check `/v1/quota/` first (and `has_valid_salesnav: true`) and surface the cost.
5. **Resolution timing + read cap.** ~90 leads resolve in ~30–60s; ~1,200 take
   several minutes; reading back early shows empty fields (`status:"pending"`) — not
   a failure. The list-read endpoint pages at ~1,000; page with an offset to verify
   larger lists. Insert in batches (~400) and use `importDuplicates: true` for a
   complete list (else existing workspace contacts are silently skipped).
6. **Verified Nous endpoints:** `POST /v2/dedup`, `POST /api/lead-lists`,
   `PATCH /api/lead-lists/<id>` (needs `workspaceId`), `POST /api/lead-lists/<id>/leads`
   — all confirmed live. Dedup result rows label the value under an `"email"` key
   even for `kind:"domain"` (cosmetic; key off `status`).
