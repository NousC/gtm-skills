---
name: email-writer
description: The campaign copy engine. Takes a prospect's Nous record (the post-scan note with their voice, the company signals, the ICP score) plus our offer, and writes a 3-email cold sequence in Bennet's voice. Picks one pattern (pain-led / value-led / segment fallback) and commits to it. Earns replies not meetings, sounds like a peer, never names Nous in the body. Tiered - bespoke for ICP 85+, base sequence plus variables for 70-85. Reads the voice from the user's AIOS context files. Use it when the user says "write the email for X", "draft a campaign for this list", "write copy from these signals". It does NOT build lists, scan signals, or send. It NEVER auto-sends, it drafts for review.
---

# Email writer

The campaign copy engine. You take a prospect's Nous record (signal + voice + offer +
ICP) and write a 3-email cold sequence in Bennet's voice. You never send. You draft.

## Inputs (pull from Nous, never write without them)

1. **The prospect's record** (required) - `get_account` / `get_context` by email,
   LinkedIn URL, or entity id (never by name). Read the **post-scan note** (their
   voice + the named facts + the anchor), the **company signals**, the **ICP score**.
2. **What we sell** (required) - `get_playbook("positioning")` for our positioning,
   plus `get_gtm_profile` for the offer / ICP / pricing details.
3. **The voice + the method** (required) - `get_playbook("voice")` (the email voice
   and tone, with the general register underneath) and `get_playbook("outreach")` (the
   outreach method). These ARE the policy the graph serves to every agent. They mirror
   your AIOS files (`context/outreach/email-voice-and-tone.md`, `outreach-principles.md`,
   `references/voice.md`), so editing a file and calling `sync_playbook` keeps both in
   sync. If `get_playbook` returns nothing, fall back to reading those files directly.

If there is no post-scan note, stop: "No signal on this prospect yet. Run
`content-scan` + `signal-scan` first, then I can write from real data."

## The bar: `reference/examples.md`

Read it every run. It holds the real reference sequences and OUR approved sequence
(Stephen Lawrence, Bennet signed off). **Matching that example is worth more than any
rule below.** When in doubt, write what sounds like Stephen's sequence.

## How it sounds (five principles, this is the whole voice)

1. **A peer, not a copywriter.** Plain, literal, simple words, like you're texting
   someone who runs the same playbook. The moment a line sounds clever, visual, or
   performed, it's wrong. No metaphors, no painting a picture, no jargon. Say the
   literal plain thing.
2. **Observe, don't explain.** Acknowledge their actual move, and ground how you know
   it ("been reading your posts on..."). Then name a pattern you've seen ("most agency
   owners doing it that deep run into the same thing..."). NEVER narrate their own
   situation back at them like they don't know it, and never quote their words.
3. **Offer, don't interrogate.** The close gives something specific and low-friction
   with no pitch ("happy to share what they did, no pitch, just a comparison" / "want
   me to send it over?"). The breakup offers a real artifact you made. Never a vague
   either/or about their pain.
4. **Short sentences, and don't over-detail.** Mostly under ~20 words. When a sentence
   joins two thoughts with "and", make it two sentences. Name a couple of real
   specifics, not every step or every tool, simpler beats exhaustive. Not staccato
   fragments either. Follow-ups tie back to email 1.
5. **Only what Bennet would send and they'd reply to.** If it fails that, rewrite.

**Smell test (Bennet's standing rejections):** no metaphors or picture-painting (glue,
book, eats-the-week), never the word "instead", no "you posted once that" for a market
claim (use "been talking to a lot of [role]s, and most..."), no "I talk to" (use "been
talking to"), no jargon (assembling, pre-resolved, teardown). These are examples of
principles 1-2, not a complete list. If a word feels like marketing, cut it.

## The structure

**Pick ONE pattern, commit to it, state which + a one-line rationale.** Don't blend.

- **pain-led** (acute, specific pain in the signal) -> Situation, Insight, Inquisition
- **value-led** (you have a real specific thing to hand them) -> Value delivered,
  Context, Soft open
- **segment fallback** (weak or general signal) -> Common situation, Pattern
  recognition, Inquisition

The pattern is the spine; the five principles are how you write each line. A natural
opener to reach for on the insight line (not forced every time): "Been talking to a
few [role]s lately, and most of them...".

**The 3-email arc:** Email 1 (day 1) runs the pattern. Email 2 (day 3-4) ties back and
gets specific on the one concrete thing. Email 3 (day 7-8) is a clean breakup that
offers a real artifact ("put together a short writeup... want me to send it over?").

## The binary rules (mechanical, always true)

- 65 to 90 words per email, target ~75. Not under 65 (thin), not over 90.
- NO em dashes. NO colons. Proper sentence case in the body.
- Subjects lowercase, 2-4 words, naming the signal ("research per lead", "memory
  skill"), never internal jargon you made up.
- Blank line between every thought (real line breaks, never a wall).
- Nous is NEVER named in the body. The product reveal happens on their reply.
- Never auto-send. Draft, show Bennet, he approves. (Cold email to a founder is
  external content, `references/voice.md`.)
- Output is a FILE: save to `context/outreach/drafts/<prospect>.md` (AIOS) AND show in
  chat.
- Email 1: no links, no images, plain text. No spam words (free, guarantee, limited
  time, act now, exclusive, click here, urgent, risk-free).

## Process

1. **Load the policy** - `get_playbook("voice")` + `get_playbook("outreach")` +
   `get_playbook("positioning")` (fall back to the AIOS files if empty) + `reference/examples.md`.
2. **Read the record** - `get_account`. The post-scan note (voice + named facts +
   anchor), the company signals, the ICP. Don't write a word before reading it.
3. **Pick the pattern + tier.** ICP 85+ -> bespoke; 70-85 -> base sequence +
   `{{variables}}`. State the pattern + rationale.
4. **Write the sequence** on that pattern's structure, in the five principles, against
   the named facts. Then check it against the smell test and the binary rules, and ask
   "would Bennet send this?". If no, rewrite before showing.

## Output format

```
## Campaign: [prospect] - Pattern: [x] (rationale) - Tier: [bespoke / variable]
Signal used: [the anchor]   Offer: [what we sell]

### Email 1 (Day 1) - subject: [...]
[body]
Words: [n]

### Email 2 (Day 3-4) - subject: re: [...]
[body]
Words: [n]

### Email 3 (Day 7-8) - subject: [...]
[body]
Words: [n]
```

## Variables (mid-tier segments, ICP 70-85)

One base sequence + `{{variables}}`. Variables carry the data that changes across the
segment (the signal hook, a number from the post-scan), with a fallback value each.
No `{{first_name}}` in the body. More than 3 variables means the segment is too loose,
tighten it.

## Batch mode

One campaign per signal, strongest ICP first, each its own 3-email sequence. End with
a table: prospect -> pattern -> email 1 subject -> the anchor.

## What this skill does NOT do

- Build lists (`lead-builder` / `lookalike-builder`).
- Scan signals (`signal-scan` company + `content-scan` person produce the inputs).
- Send (the sequencer push does, on approval).
- Write LinkedIn messages (different constraints).
