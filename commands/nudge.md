# /nudge — Follow-Up Tracker

## Description

Surface threads where you sent the last message and haven't heard back.
Draft follow-up nudges in your voice for review and sending.

## Arguments

- (none) — Full scan of Gmail and WhatsApp for stale threads
- `gmail` — Only scan Gmail
- `whatsapp` — Only scan WhatsApp

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

### Step 1: Scan for Stale Threads

**Gmail (if enabled):**
1. Search for sent emails from the last 30 days
2. For each sent email, check if a reply exists in the thread
3. If no reply and the sent date is > `default_window_days` ago, flag as stale

**WhatsApp (if enabled):**
1. Check recent WhatsApp conversations
2. Identify threads where your message is the most recent
3. If the last message was > `default_window_days` ago, flag as stale

### Step 2: Filter Results

Remove from the stale list:
- Threads matching any `ignore_patterns` (noreply, newsletters, etc.)
- Threads where your last message was a closing statement ("Thanks", "Got it",
  "Sounds good", "Will do", or similar — messages that don't expect a reply)
- Threads already in `nudge-log.yaml` where:
  - `nudge_count` >= `max_nudges_per_contact`, OR
  - `last_nudge` was < `nudge_cooldown_days` ago

### Step 3: Present Stale Threads

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

### Step 4: Draft Nudges

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

### Step 5: Update Nudge Log

After each nudge is sent (or saved as Gmail draft):

Update `~/.claude/nudge-log.yaml`:
- If thread already exists in log: increment `nudge_count`, update `last_nudge`
- If new thread: add entry with `nudge_count: 1` and today's date

### Step 6: Summary

```
NUDGE SUMMARY

Sent: [count]
- [NAME] — [SUBJECT] (via [channel])
- ...

Saved as Gmail draft: [count]
- [NAME] — [SUBJECT]

Skipped: [count]

Next suggested nudge check: [today + default_window_days]
```

### Guidelines

- Keep nudges SHORT. A nudge is not a new email — it's a gentle follow-up.
- Never nudge more than `max_nudges_per_contact` times on the same thread.
- If a thread has been nudged twice with no reply, it's probably dead. Don't push further.
- The tone should always be warm and understanding — people are busy.
- When in doubt about whether a thread needs a nudge, include it in the list
  and let the user decide (they can always skip).
