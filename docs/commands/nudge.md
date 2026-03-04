# /nudge — Follow-Up Tracker

## Description

Surface follow-ups driven by your Notion CRM pipeline, enriched with Gmail context.
Also scans for stale threads where you sent the last message and haven't heard back.
Draft follow-up nudges in your voice for review and sending.

## Arguments

- (none) — CRM-driven nudges (primary) + stale thread scan (secondary)
- `crm` — Only CRM-driven nudges
- `gmail` — Only scan Gmail for stale threads
- `whatsapp` — Only scan WhatsApp for stale threads

## Instructions

You are running the nudge tracker. Follow these steps in order.

### Step 0: Load Configuration

Read `~/.claude/nudge-config.yaml` for settings:
- `default_window_days`: how many days without reply = stale
- `channels`: which channels to scan
- `ignore_patterns`: email patterns to skip
- `max_nudges_per_contact`: cap on nudges per thread
- `nudge_cooldown_days`: minimum days between nudges

Read `~/.claude/nudge-log.yaml` for nudge history.

---

## PART A: CRM-Driven Nudges (Primary)

These are the highest-priority nudges. They come from the Notion CRM pipeline
and represent scheduled follow-ups with prospects and customers.

### Step 1A: Fetch All Active CRM Entries

Query the Notion database with ID `{{CRM_DATABASE_ID}}`.
Fetch ALL pages in the database — do not filter by keyword or rely on semantic search.
Use pagination to ensure you retrieve every record.

### Step 2A: Filter by Today's Next Step Date

For each page returned, read the `Next Step Date` property.
Keep only records where `Next Step Date.start` exactly matches today's date
(use today's date in YYYY-MM-DD format).

If no records match today, report "No CRM nudges due today" and proceed to Part B.

### Step 3A: Enrich Each Matching Record

For each matching prospect/customer:

a) **Read the full Notion page** to extract all available context:
   company name, contact name, deal stage, notes, last interaction, any stated next steps.

b) **Search Gmail** for recent threads with this contact or their company.
   Look for the most recent email exchange, any outstanding questions,
   commitments made, or proposals sent.

c) **Query the Document Hub** (database ID `{{DOCUMENT_HUB_DATABASE_ID}}`)
   for meeting notes and documents linked to this customer.
   The `Customer` relation field cross-references the CRM database.
   Filter by the matching customer relation, then read the most recent
   documents to extract relevant context (discussion points, decisions,
   commitments, action items).

d) Note the **current deal stage** and any ROI figures or case studies
   that might be relevant. Stackfix case studies for reference:
   - Caterparts: 8.6x ROI
   - Motormax: 10x+ ROI
   - Medical Law Partnership: 9-13x ROI

### Step 4A: Draft CRM Nudge Emails

For each enriched record:

1. **Load voice examples** from `~/.claude/voice/`:
   - Prioritize pairs with the same recipient
   - Then pairs with similar topics (sales, follow-up, outreach)
   - Include 3-5 relevant pairs as style references

2. **Draft a tailored outreach email** that:
   - Uses all gathered context (Notion notes, Gmail history, meeting notes)
   - References the specific next step or commitment from the CRM record
   - Is personalized — not generic. Mention specifics from prior conversations.
   - Sounds like the user's voice (use voice/ examples)
   - Is warm, direct, and action-oriented
   - Includes a clear ask or proposed next step
   - Where relevant, weaves in ROI data or case studies naturally (don't force it)

3. **Present using the draft-edit-send loop:**

```
---
CRM NUDGE: [COMPANY NAME] — [CONTACT NAME]
Deal Stage: [STAGE] | Next Step Date: today
Context: [1-line summary of what the next step is about]

Draft:
> [The email draft]

Actions: [s]end | [e]dit | [skip] | [r]egenerate | [g]mail draft
---
```

Handle actions identically to the /triage draft-edit-send loop:
- [s]end: send via Gmail MCP, no voice pair saved
- [e]dit: user edits, send, save voice pair
- [skip]: move to next
- [r]egenerate: new draft with different approach
- [g]mail draft: save as Gmail draft

After each CRM nudge is sent or saved, update the Notion page's `Next Step Date`
if the user specifies a new follow-up date.

---

## PART B: Stale Thread Scan (Secondary)

These are opportunistic nudges — threads where you're waiting on a reply.
Only run this part if the argument is not `crm`.

### Step 1B: Scan for Stale Threads

**Gmail (if enabled):**
1. Search for sent emails from the last 30 days
2. For each sent email, check if a reply exists in the thread
3. If no reply and the sent date is > `default_window_days` ago, flag as stale

**WhatsApp (if enabled):**
1. Check recent WhatsApp conversations
2. Identify threads where your message is the most recent
3. If the last message was > `default_window_days` ago, flag as stale

### Step 2B: Filter Results

Remove from the stale list:
- Threads matching any `ignore_patterns` (noreply, newsletters, etc.)
- Threads where your last message was a closing statement ("Thanks", "Got it",
  "Sounds good", "Will do", or similar — messages that don't expect a reply)
- Threads already handled in Part A (same contact, same topic)
- Threads already in `nudge-log.yaml` where:
  - `nudge_count` >= `max_nudges_per_contact`, OR
  - `last_nudge` was < `nudge_cooldown_days` ago

### Step 3B: Present Stale Threads

```
STALE THREADS ([count] found)

1. [NAME] ([Xd] ago) — [SUBJECT/TOPIC]
   Channel: [gmail/whatsapp]
   Your last message: "[First 80 chars of your sent message...]"

2. [NAME] ([Xd] ago) — [SUBJECT/TOPIC]
   Channel: [gmail/whatsapp]
   Your last message: "[First 80 chars...]"

...

Select threads to nudge (e.g., "1,3" or "all") or "skip" to exit:
```

Sort by days waiting (longest first).

### Step 4B: Draft Stale Thread Nudges

For each selected thread:

1. **Load voice examples** from `~/.claude/voice/`:
   - Prioritize pairs with the same recipient
   - Then pairs with similar topics
   - Include 3-5 relevant pairs as style references

2. **Read the full thread** for context (what you asked, what the topic is)

3. **Draft a nudge** that:
   - Is brief (2-4 sentences max)
   - References the original topic naturally
   - Sounds like the user's voice (use voice/ examples)
   - Is warm and not aggressive — "circling back" not "per my last email"
   - Includes a clear ask or next step

4. **Present using the draft-edit-send loop:**

```
---
[CHANNEL] Nudge to [NAME] — [SUBJECT/TOPIC]
Original sent [X days] ago | Nudge #[N] of max [max_nudges_per_contact]

Draft:
> [The nudge draft]

Actions: [s]end | [e]dit | [skip] | [r]egenerate | [g]mail draft
---
```

Handle actions identically to the /triage draft-edit-send loop:
- [s]end: send via MCP, no voice pair saved
- [e]dit: user edits, send, save voice pair
- [skip]: move to next
- [r]egenerate: new draft
- [g]mail draft: save as Gmail draft (Gmail only)

---

## Step 5: Update Nudge Log

After each nudge is sent (or saved as Gmail draft), for both Part A and Part B:

Update `~/.claude/nudge-log.yaml`:
- If thread already exists in log: increment `nudge_count`, update `last_nudge`
- If new thread: add entry with `nudge_count: 1` and today's date

## Step 6: Summary

```
NUDGE SUMMARY

CRM Nudges (due today): [count]
- [COMPANY] / [CONTACT] — [STAGE] (sent / drafted / skipped)
- ...

Stale Thread Nudges: [count]
- [NAME] — [SUBJECT] (via [channel]) (sent / drafted / skipped)
- ...

Next suggested nudge check: [tomorrow]
```

## Guidelines

- **CRM nudges take priority** — always process Part A first.
- CRM nudges can be longer and more tailored than stale thread nudges.
- Stale thread nudges should be SHORT (2-4 sentences). A nudge is not a new email.
- Never nudge more than `max_nudges_per_contact` times on the same thread.
- If a thread has been nudged twice with no reply, it's probably dead. Don't push further.
- The tone should always be warm and understanding — people are busy.
- When in doubt about whether a thread needs a nudge, include it in the list
  and let the user decide (they can always skip).
- When weaving in case studies or ROI figures, do so naturally — don't make it feel like a pitch deck.
