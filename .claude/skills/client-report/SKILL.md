---
name: client-report
description: Write a weekly, client-ready report for the outbound you run for a client. It reads everything in that client's Nous workspace — the lead lists you sourced and sent, the People who engaged and advanced, the funnel, and which messages and segments are converting — and writes a polished report into a document you can hand to the client. Built for agencies running a workspace per client.
---

# Weekly Client Report

## What it does

Every week, it writes the report you'd otherwise build by hand. It reads the
whole picture for one client from their Nous workspace — the **lead lists** you
sourced and sent, the **People** who engaged and advanced a stage, the **funnel**
(replies, meetings, conversions), and **which messages and segments are actually
converting** — and writes a clean, client-facing report into a document you can
send as-is. Grounded in real data, no number invented, ending in what you'll do
next week.

The flow: **Nous → Claude Code → a document.**

## How to invoke

`/client-report` — or *"write this week's report for Acme."*

## First-run setup (you, the agent, run this once as a short interview)

Detect what's connected, ask only for what's missing, one thing at a time.

**1. Nous — the client's workspace.** Check whether the Nous MCP is connected
(try the `query` tool).
- Connected → continue.
- Not → "The report reads from Nous. Connect it in one line:
  `claude mcp add nous -e NOUS_API_KEY=<your-key> -- npx -y @opennous/mcp`. Use
  the API key for **this client's workspace** (opennous.cloud → Settings → API
  keys, the `pk_` one) — one workspace per client keeps the data clean."
  The lead-list and copy reads also use `NOUS_API_KEY` over REST, so set it in
  the env too.

**2. Where should the report go?** Ask: "Google Docs, Notion, or a markdown file?
Attach the connector for the one you want and I'll write the report there.
Otherwise I'll hand you the markdown to paste."

**3. Which client and which week?** Confirm the workspace and the period
(default the last 7 days).

## The process

Read every surface, then write one report. Count from Nous; never fabricate.

### 1. Scope the client and the period

Confirm the workspace (the client) and the window (default 7 days). Every number
below is for that workspace, that window.

### 2. Read the lead lists — what you sourced and sent

```
GET /api/lead-lists?workspaceId=<ws>          # the lists, with counts + source
GET /api/lead-lists/<id>/leads?workspaceId=<ws>
```

Capture: new leads added this week (by list / source), and how many were
suppressed as already-touched (the spend you saved them).

### 3. Read the People who engaged — who moved

```
query { scope: { property: 'interaction.positive_reply', since_days: 7 }, return: 'entities' }
query { scope: { property: 'interaction.meeting_held',    since_days: 7 }, return: 'entities' }
```

Each entity carries its `firmographics` (industry, title) and
`most_recent_attribution` (the variant that earned the reply). These are the
people who replied, advanced a stage, or booked. Pull the full record on the
notable ones with `get_account`.

### 4. Read the funnel — the rates

```
query { scope: { property: 'interaction.email_sent',     since_days: 7 }, return: 'entities' }
query { scope: { property: 'interaction.email_received', since_days: 7 }, return: 'entities' }
```

Compute: sent → replied → positive → meeting, and the reply rate **by variant**
and **by industry** (group the step-3/4 results by `attribution` and
`firmographics`). Read the converting copy:

```
GET /api/campaign-messages?workspaceId=<ws>&campaignId=<id>
```

### 5. Read the conversations — wins + voice of customer

The most signal-rich positive replies, verbatim, as the client-facing proof.
Use the `summary` on the reply observations.

### 6. Write the report (client-facing)

Polished, no internal jargon, one page where possible:

```
# <Client> — week of <date range>

## This week at a glance
  Leads added, sent, replies, positive replies, meetings booked. One line each.

## What we did
  Lists and campaigns run this week, with volume.

## What's working
  The top converting segment(s) and message(s), with the winning copy.

## Conversations + wins
  Meetings booked, notable replies, voice of customer (verbatim).

## The funnel
  sent → replied → positive → meeting, with the rate. The one number to watch.

## Next week
  What we'll double down on, and what we'll change.
```

### 7. Save it

Write the report to the document the operator chose (Google Docs / Notion), or
hand back the markdown. Return the link.

## Hard rules — never break these

- **Every number comes from Nous.** Count it; never estimate or invent.
- **Flag thin data.** A segment with under ~20 sends is "not enough data yet",
  not a conclusion.
- **Client-facing tone.** No internal jargon, no raw tool names the client
  doesn't know. It's a report you'd send without editing.
- **Always end in next week.** A report without a recommendation is a data dump.

## Customize / Set up

- **Schedule it weekly** — run every Monday morning so the report is ready to send.
- **Pick the document target** — Google Docs, Notion, or a markdown file.
- **Scope it** — limit to specific lists or campaigns, or set the period.
- **Match the client's voice** — keep a short brief per client (tone, what they
  care about) so each report reads like it was written for them.

## Frequently asked questions

**Does it work across multiple clients?**
One report per client workspace. Run it once per client (each with its own
`pk_` key), or point it at each workspace in turn.

**What if a section has no data this week?**
It says so plainly ("no meetings booked this week") rather than padding. Honesty
reads as trust in a client report.

**Can the client see the underlying data?**
The report is the deliverable. If they want to dig in, that's a Nous workspace
seat, not this skill.
