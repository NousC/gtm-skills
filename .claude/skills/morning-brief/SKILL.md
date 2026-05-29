---
name: morning-brief
description: Every morning, read your goals and milestones from Timestripe and your GTM funnel from Nous, then write one brief — what to work first today tied to your goals, where the funnel stands (momentum, conversations, follow-ups, cooled accounts, voice of customer), the roadmap anchor, and one or more 1% Kaizen improvements saved to Notion. Keeps you accountable to the day, the week, and the roadmap.
---

# Morning Brief

## What it does

Your morning coffee, written for you. It reads your **goals and milestones from
Timestripe**, your **GTM funnel from Nous** (the last 7 days of momentum,
conversations, follow-ups, cooled accounts, and voice of customer), and your
roadmap, then writes one brief: **what to work first today** tied to today's and
this week's goals, where the funnel stands, and **one or more 1% Kaizen
improvements** saved to Notion. It keeps you accountable to the day, the week,
and the quarter — and gets 1% sharper at running you every day.

The flow: **Timestripe → Nous → Claude Code → Notion.**

## How to invoke

`/morning-brief` — or schedule it to run every morning (see Customize).

## Setup

- **Nous MCP** attached (`query`, `account`, `attention`) — the GTM funnel.
- **Timestripe** access (CLI or API key) — day goals, week goals, milestones.
- **Notion** connector — the Kaizen database the brief writes to.
- A **`config.md`** in the skill's folder — who and what to filter out
  (recurring meetings, non-prospect contacts, vendor cold-pitches) so the funnel
  reads clean. The skill tunes this over time from the "open issues" it logs.

## The 1% Kaizen idea — the heart of it

Each morning the brief proposes **one to three tiny improvements**, every one
**triggered by something real in today's data** — a goal dropped silently
yesterday, a meeting confirmation you left unacknowledged, a launch bucket
that's scattered. Each Kaizen carries a `trigger` (the evidence) and a
`category` (focus / responsiveness / …), and is saved to your Notion Kaizen DB
as `Status=proposed` for review. Never generic advice. 1% better every day,
compounding, on the record.

## The process

### 1. Read your goals (Timestripe)

Pull today's **day goals**, this week's **week goals** (with due dates), the
open **milestones** per bucket, and the **roadmap anchor** (the quarter's north
stars). Note what's done, what's unchecked, what's past-due, and what carried
over from yesterday versus what was silently dropped.

### 2. Read your funnel (Nous, last 7 days)

Honoring `config.md` filters, pull from Nous:

- **Momentum** — sales meetings (recurring ones filtered), new connects, replies.
- **Active conversations**, parsed by direction — who's awaiting *your* reply,
  who you're awaiting, the last thing said.
- **Follow-up queue** — stale outbound in the window; optionally a weekly 14-day
  long-stale sweep.
- **Cooled / attention** — accounts gone quiet, via the Nous `attention` tool.
- **Voice of customer** — the most signal-rich inbound replies, verbatim.

### 3. Decide what to work first

Cross the funnel against the goals and surface **2-3 prioritized actions** for
today — each tied to a specific goal or a specific person. The momentum-
preserving 30-second reply ranks next to the P1 launch deliverable.

### 4. Propose the 1% Kaizen

From today's data, propose 1-3 Kaizen improvements (`trigger` + `category`) and
save them to the Notion Kaizen DB as `Status=proposed`. Return a link.

### 5. Deliver the brief

Write the full morning coffee, in this order:

```
## What I'd work first today        — 2-3 prioritized actions, each tied to a goal/person
## Today + this week (Timestripe)    — day goals, week goals, open milestones
## Last 7 days in the funnel (Nous)  — momentum, conversations, follow-ups, cooled, voice of customer
## Roadmap anchor                    — the quarter's north stars + status
## 1% Kaizen — proposed today        — the improvements, + the Notion link
## Open issues (for config tuning)   — filters to add, edge cases the brief found
```

## Hard rules — never break these

- **Every Kaizen is triggered by real data.** Cite the trigger. No generic
  "be more productive" advice.
- **Honor `config.md`.** Recurring meetings, non-prospects, and vendor pitches
  stay out of the funnel read.
- **Never fabricate funnel numbers.** Count from Nous; if a section is empty,
  say so.
- **Accountability, not noise.** Surface what's past-due and what was dropped —
  the brief's job is to keep you honest about the week.

## Customize / Set up

- **Schedule it** — run every morning at 9am as a routine
  (`/schedule morning-brief daily at 9am`) or a cron job, so it's waiting when
  you sit down.
- **Tune `config.md`** — add people and senders to exclude; mark which meetings
  are recurring. The brief's "open issues" section tells you what to add.
- **Point it at your Kaizen DB** — set the Notion database id for the Kaizen log.
- **Funnel window** — 7 days by default; turn on a weekly 14-day long-stale
  sweep for the accounts aging out of the window.
- **Dry-run** — generate the brief without writing the Kaizen to Notion while
  you tune it.

## Frequently asked questions

**Why three tools?**
Timestripe holds the goals you're accountable to, Nous holds the GTM funnel
that feeds them, and Notion is where the Kaizen log lives and compounds. The
brief is the one place they meet each morning.

**What's a Kaizen, concretely?**
A one-line improvement triggered by today's data — e.g. *"Acknowledge meeting
confirmations within 4 hours"* because a confirmation sat unanswered for a day.
Small, specific, logged, reviewed.

**Does it act on anything, or just report?**
It reports and proposes. The only thing it writes is the Kaizen log to Notion
(as `proposed`). The actions are yours to take.
