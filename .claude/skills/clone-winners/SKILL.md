---
name: clone-winners
description: Turn your closed-won deals into your next campaign. It pulls the deals that actually closed from Nous, derives the revenue-validated ICP from them, finds lookalike companies (DiscoLike), enriches the buyers (Apollo + FullEnrich), analyzes which sequence actually won those deals, and writes a new campaign in that winning pattern — the leads and the copy, saved to Nous. The closed-won flywheel: clone who pays you, and reach them the way that won them. Needs closed-won history to seed from.
---

# Clone winners

## What it does

The strongest signal in GTM isn't a reply, it's a **close**. This skill takes
the deals that actually closed, finds more companies like them, and writes the
campaign in the pattern that won them. One run gives you both halves: a
**lookalike lead list** of your best-fit ICP, and a **sequence** built from what
actually closed similar deals — not a template, not just what got a reply.

The flywheel: **closed deals → lookalikes → winning campaign → more closed deals.**

The flow: **Nous → DiscoLike → Apollo → Nous.**

## How to invoke

`/clone-winners` — or *"clone my last 5 closed deals into a new campaign."*

## First-run setup (you, the agent, run this once as a short interview)

This skill builds on `lead-builder` (discovery + enrichment) and `campaign-writer`
(copy), so it needs the same keys, plus closed-won history in Nous.

**1. Nous** — the closed deals, the ICP, and what won them. Connect the MCP
(`get_account`, `get_gtm_profile`, `query`, `record`) and set `NOUS_API_KEY`
(`pk_`) for the REST calls. Not connected →
`claude mcp add nous -e NOUS_API_KEY=<key> -- npx -y @opennous/mcp`.

**2. DiscoLike** (`DISCOLIKE_API_KEY`) — lookalike company discovery, seeded by
your closed-deal domains. It's a subscription that converts to credits, from
**$99/mo (Starter)**, drawn down at **$3.50 per 1,000 new companies** (less on
higher tiers); records cache 90 days, so repeats are free.

**3. Apollo** (`APOLLO_API_KEY`, master key) — the buyer + email. Optional
`FULLENRICH_API_KEY` (waterfall) and `MILLIONVERIFIER_API_KEY` (verify).

**Prerequisite:** you need **closed-won deals** in Nous (the `client` stage). No
closes yet → use `campaign-writer` instead (it learns from replies, which come
earlier).

## Core philosophy

**Closed-won is the gold signal.** A reply means curious; a close means they
paid. Seed from revenue, not from a description you typed.

**Clone who pays you.** The best-fit ICP is whoever actually closed — let the
wins define the target, then find more of them.

**Reach them the way that won them.** Don't guess the angle; read the sequence
that closed similar deals and write in that pattern.

**Show the cost before you spend, dedup before you enrich** — same discipline as
`lead-builder`.

## The process

### 1. Pull the closed-won deals

```bash
curl -s -X POST "https://api.opennous.cloud/v2/query" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "scope": { "kind": "state", "property": "pipeline_stage", "since_days": 365 },
        "return": "entities" }'
```

Keep the entities whose `most_recent_value` is `client` (the won stage). Each
carries `firmographics` (industry, size, title). For each, `get_account` to get
the **company domain** — those domains are your lookalike seed. If the operator
named a specific set ("my last 5"), use those.

### 2. Derive the proven ICP, cross-check the saved one

From the closed deals' firmographics, name the revenue-validated ICP — the
industry, size, and tech they share. Then pull the *stated* ICP
(`get_gtm_profile` / `GET /v2/workspace/facts?categories=ICP`) and compare:

> "Your 5 closed deals are **B2B fintech, 50–200, on Stripe + Segment** — your
> saved ICP says **'SaaS, 20–500'**. The wins are tighter. Clone the wins, or
> widen to your saved ICP?"

The wins are usually sharper than the stated ICP. Let the operator confirm.

### 3. Analyze what *won* them

For each closed account, read its outbound timeline and find the sequence that
earned the reply that led to the close:

```bash
curl -s -X POST "https://api.opennous.cloud/v2/query" \
  -H "Authorization: Bearer $NOUS_API_KEY" -H "Content-Type: application/json" \
  -d '{ "scope": { "entity_id": "<closed_account_id>", "property": "interaction.email" },
        "return": "observations" }'
```

Each `interaction.email_sent` carries the `attribution` (campaign / step /
variant / subject); the `interaction.email_received` is the reply. Find the
common winning thread across the closed deals — the angle, the opener, the CTA.
Read the actual copy:

```bash
curl -s "https://api.opennous.cloud/api/campaign-messages?workspaceId=<ws>&campaignId=<id>" \
  -H "Authorization: Bearer $NOUS_API_KEY"
```

This is the pattern you'll write in — proven by revenue, not by a reply.

### 4. Confirm the plan and a rough cost, then wait

```
Cloning: 5 closed deals (B2B fintech, 50–200, Stripe + Segment)
Lookalikes: ~2,000 companies → ~1,400 net-new after dedup
Winning angle: "[the opener/angle that closed the seed deals]"
Rough cost: ~$7 discovery + ~$70 enrichment ≈ $77
Run it?
```

Never spend before a yes.

### 5. Find lookalikes, dedup, enrich  (the lead-builder pipeline)

Seed DiscoLike with the closed-deal domains, dedup the results against Nous by
domain, and enrich the buyers — exactly the `lead-builder` flow:

```bash
curl -s "https://api.discolike.com/v1/discover?domain=closed1.com&domain=closed2.com&employee_range=50,200&max_records=2000" \
  -H "x-discolike-key: $DISCOLIKE_API_KEY"
# → /v2/dedup by domain (keep net_new) → Apollo api_search + bulk_match
#   → FullEnrich on misses → MillionVerifier
```

### 6. Write the winning-pattern campaign

Write a sequence for the lookalikes in the pattern that **closed** the seed
deals — the proven angle, opener, and CTA from step 3, grounded in your GTM
profile, with per-lead variables. This is `campaign-writer`'s job, anchored on
*what won* instead of *what replied*. Always include a fallback angle.

### 7. Save and report

Create a Nous lead list (default name `Lookalikes of <N> closed deals · <ICP>`),
insert the enriched buyers, and record the campaign copy per variant via
`POST /api/campaign-messages` so the loop keeps learning. Then:

> "Done. ~1,400 lookalikes of your 5 closed deals are landing in your new list,
> with a 3-email sequence written in the angle that closed them. Push it to your
> sequencer, or refine it with `campaign-writer`."

## Hard rules — never break these

- **Seed from real closes.** If there are no closed-won deals, stop and point to
  `campaign-writer` — don't fabricate a winning pattern.
- **The wins define the ICP.** Surface where the closed deals diverge from the
  saved ICP and let the operator choose.
- **Cost + approval before spend; dedup before enrich.**
- **Write from what closed, cite it.** The angle traces to the sequence that won
  the seed deals, not a guess.

## Customize / Set up

- **Pick the seed** — your last N closes, a date range, or a named set.
- **Clone tight or widen** — match the closed deals exactly, or relax toward the
  saved ICP.
- **Match rate vs cost** — Apollo-only, or add FullEnrich + MillionVerifier.
- **Hand off** — push the campaign to your sequencer, or open it in
  `campaign-writer` to iterate.

## Frequently asked questions

**How is this different from lead-builder + campaign-writer?**
Those start from a description you type. This starts from your **wins** — it
derives the ICP and the winning angle from deals that actually closed, then runs
both halves automatically. It's the two skills, seeded by revenue.

**What if I have very few closed deals?**
Even 3–5 is enough to seed a lookalike and read a winning angle. With none, use
`campaign-writer` (reply-based) until the closes accumulate.

**Does it learn over time?**
Yes. It records the new campaign's copy back to Nous, so as the lookalikes
convert, the next run clones a bigger, sharper set of winners.
