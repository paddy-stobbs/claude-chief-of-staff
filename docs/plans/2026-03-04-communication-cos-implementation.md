# Communication Chief of Staff — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fork claude-chief-of-staff and extend it with voice learning, a draft-edit-send loop, and a nudge tracker for Gmail + WhatsApp.

**Architecture:** Fork the existing repo, keep all commands/config intact, add a `voice/` directory for style learning via few-shot pairs, modify `/triage` to include a draft-edit-send review loop, and add a new `/nudge` command for follow-up tracking.

**Tech Stack:** Claude Code CLI, MCP servers (Gmail, WhatsApp, Google Calendar), YAML config, Markdown command definitions.

---

### Task 1: Fork repo and run initial setup

**Files:**
- Clone: `claude-chief-of-staff` repo into `~/chief-of-staff/`
- Run: `install.sh`

**Step 1: Fork and clone the repo**

```bash
cd ~
# Remove the placeholder repo we created during design
rm -rf ~/chief-of-staff
# Fork on GitHub (or just clone directly to start)
gh repo fork mimurchison/claude-chief-of-staff --clone=true --remote=true
# Rename to chief-of-staff for convenience
mv claude-chief-of-staff chief-of-staff
cd chief-of-staff
```

**Step 2: Run the installer**

```bash
cd ~/chief-of-staff
bash install.sh
```

This will prompt for your name, role, company, email, timezone, etc. It copies files to `~/.claude/` and personalizes CLAUDE.md.

**Step 3: Configure MCP servers**

Ensure these MCP servers are connected in Claude Code settings:
- Gmail MCP (required)
- WhatsApp MCP (required)
- Google Calendar MCP (recommended)

Verify by running Claude Code and checking available tools. You should see Gmail and WhatsApp tools listed.

**Step 4: Verify base system works**

Run `/gm` in Claude Code. Expected: a morning briefing with calendar, tasks, and inbox scan. If Gmail MCP isn't connected, you'll see an error about missing tools.

**Step 5: Commit baseline**

```bash
cd ~/chief-of-staff
git add -A
git commit -m "feat: initial fork with personalized config"
```

---

### Task 2: Create voice learning directory and README

**Files:**
- Create: `voice/README.md`
- Create: `voice/.gitkeep`

**Step 1: Create the voice directory**

```bash
mkdir -p ~/chief-of-staff/voice
```

**Step 2: Write the voice README**

Create `voice/README.md` with this content:

```markdown
# Voice Learning Directory

This directory stores draft-vs-edit pairs that teach the system your writing style.

## How it works

When you edit a draft before sending, the system saves the original draft and your
edited version as a YAML file. Over time, these pairs are used as few-shot examples
to make future drafts sound more like you.

## File format

Each file is a YAML document with this structure:

    timestamp: 2026-03-04T09:15:00
    context: "Brief description of what the email was about"
    channel: gmail | whatsapp
    recipient: "Name of the recipient"
    original_draft: |
      The system-generated draft text...
    edited_version: |
      Your edited version of the draft...
    tags: [topic1, topic2, tone-descriptor]

## Bootstrapping

To seed the system with your voice before any edits accumulate, create files
with only `edited_version` (no `original_draft`). Paste sent emails you're
proud of as examples of your natural writing style.

## Selection

When drafting a new response, the system scans this directory for the 3-5 most
relevant pairs based on: same recipient, similar topic/tags, same channel.
```

**Step 3: Add .gitkeep for empty directory tracking**

```bash
touch ~/chief-of-staff/voice/.gitkeep
```

**Step 4: Commit**

```bash
cd ~/chief-of-staff
git add voice/
git commit -m "feat: add voice learning directory with README"
```

---

### Task 3: Update CLAUDE.md with voice learning instructions

**Files:**
- Modify: `~/.claude/CLAUDE.md` (the installed copy)
- Modify: `~/chief-of-staff/CLAUDE.md` (the repo template)

The voice learning instructions need to go in two places in CLAUDE.md:
1. Part 4 (Writing Style) — tell Claude to check `voice/` for examples
2. Part 6 (Operating Modes) — add behavior for the Draft mode

**Step 1: Add voice learning section to Part 4 (Writing Style)**

In `CLAUDE.md`, after the "Example Emails" subsection and before "Scheduling in Responses", add:

```markdown
### Voice Learning

**Location:** `~/.claude/voice/`

Before drafting ANY response (email, WhatsApp, nudge), check the `voice/` directory
for relevant style examples.

**Selection process:**
1. Scan all `.yaml` files in `voice/`
2. Rank by relevance: prioritize same recipient > same tags/topic > same channel
3. Include the top 3-5 pairs in your drafting context as few-shot examples
4. Use the `edited_version` fields to calibrate tone, vocabulary, and sentence structure
5. If a pair has only `edited_version` (no `original_draft`), treat it as a pure style reference

**When saving pairs:**
- After the user edits a draft and sends it, save the pair to `voice/`
- Filename format: `YYYY-MM-DD-reply-to-<name>-<topic>.yaml`
- Auto-generate tags from the email content (recipient type, topic, tone)
- Do NOT save a pair if the user sent the draft without editing (no corrections = draft was good)
- Do NOT save if the only changes were typo fixes (< 5 characters changed)
```

**Step 2: Verify the section is in the right location**

Read CLAUDE.md and confirm the new section sits between "Example Emails" and "Scheduling in Responses" in Part 4.

**Step 3: Copy changes to repo template**

Copy the same changes into `~/chief-of-staff/CLAUDE.md` so the template includes voice learning for future setups.

**Step 4: Commit**

```bash
cd ~/chief-of-staff
git add CLAUDE.md
git commit -m "feat: add voice learning instructions to CLAUDE.md"
```

Also update the installed copy:

```bash
cp ~/chief-of-staff/CLAUDE.md ~/.claude/CLAUDE.md
```

---

### Task 4: Modify triage.md with draft-edit-send loop

**Files:**
- Modify: `~/chief-of-staff/commands/triage.md`
- Modify: `~/.claude/commands/triage.md` (installed copy)

**Step 1: Add the draft-edit-send loop section**

In `commands/triage.md`, after the existing draft generation section, add a new section for the review loop. This replaces the existing "present drafts for approval" flow with a more structured interactive loop:

```markdown
### Draft-Edit-Send Loop

After generating drafts for triaged items, present each draft using this interactive loop:

**For each draft, present:**

```
---
[CHANNEL] Reply to [SENDER] — [SUBJECT/TOPIC]
Thread context: [1-2 sentence summary of what they said]

Draft:
> [The generated draft text, using voice/ examples for style matching]

Actions: [s]end | [e]dit | [skip] | [r]egenerate | [g]mail draft
---
```

**Action handling:**

- **[s]end** — Send immediately via the channel's MCP server (Gmail or WhatsApp). Do NOT save a voice pair. Move to next item.

- **[e]dit** — Ask the user to provide their edited version. After they provide it:
  1. Show the edited version for confirmation
  2. Ask: "Send this? [y/n]"
  3. If yes: send via MCP, then save the original-vs-edited pair to `voice/` as a YAML file
  4. If no: return to the actions menu with the edited text as the new draft

- **[skip]** — Move to the next item without sending anything.

- **[r]egenerate** — Generate a new draft with a different approach. Pick different voice/ examples or adjust tone. Present the new draft with the same action menu.

- **[g]mail draft** (Gmail only) — Save as a draft in Gmail using the `gmail_create_draft` tool. Inform the user: "Draft saved in Gmail. Open Gmail to add attachments or review formatting before sending." If the user edited before choosing this option, save the voice pair. Move to next item.

**"Draft all" mode:**

If the user says "draft all" after seeing the triage summary, generate drafts for ALL actionable items (Tier 1 and Tier 2), then walk through them one by one using the loop above.

**Voice example loading:**

Before generating any draft, load relevant examples from `voice/`:
1. Check `~/.claude/voice/` for `.yaml` files
2. Find pairs matching: same recipient > same topic > same channel
3. Include top 3-5 as few-shot style references in the drafting prompt
4. If no relevant pairs exist, rely on CLAUDE.md writing style section
```

**Step 2: Ensure WhatsApp is included in the channel scan**

Check that the triage.md channel scanning section includes WhatsApp as a required channel (not optional). It should scan Gmail and WhatsApp with equal priority.

**Step 3: Copy to installed location**

```bash
cp ~/chief-of-staff/commands/triage.md ~/.claude/commands/triage.md
```

**Step 4: Test the updated triage**

Run `/triage` in Claude Code. Expected behavior:
- Scans Gmail and WhatsApp
- Presents triaged items by tier
- For each item, shows the draft-edit-send action menu
- Editing a draft and sending should create a `.yaml` file in `voice/`

**Step 5: Commit**

```bash
cd ~/chief-of-staff
git add commands/triage.md
git commit -m "feat: add draft-edit-send loop with voice capture to triage"
```

---

### Task 5: Create nudge-config.yaml

**Files:**
- Create: `~/chief-of-staff/nudge-config.yaml`
- Copy to: `~/.claude/nudge-config.yaml`

**Step 1: Create the config file**

Create `nudge-config.yaml`:

```yaml
# Nudge Tracker Configuration
# Controls how the /nudge command identifies and handles stale threads

# How many days without a reply before a thread is considered stale
default_window_days: 3

# Which channels to scan for stale threads
channels:
  gmail: true
  whatsapp: true

# Email patterns to ignore (these never need follow-ups)
ignore_patterns:
  - "noreply@"
  - "no-reply@"
  - "notifications@"
  - "donotreply@"
  - "mailer-daemon@"
  - "newsletter@"

# Maximum number of nudges to send to the same contact for the same thread
max_nudges_per_contact: 2

# Minimum days between nudges to the same thread
nudge_cooldown_days: 5
```

**Step 2: Copy to installed location**

```bash
cp ~/chief-of-staff/nudge-config.yaml ~/.claude/nudge-config.yaml
```

**Step 3: Create empty nudge log**

Create `nudge-log.yaml`:

```yaml
# Nudge Log — tracks which threads have been nudged and when
# This file is automatically updated by the /nudge command
# Do not edit manually unless correcting an error

nudges: []

# Each entry has this format:
# - thread_id: "<gmail thread ID or whatsapp thread reference>"
#   channel: gmail | whatsapp
#   recipient: "Name"
#   first_nudge: 2026-03-04
#   last_nudge: 2026-03-04
#   nudge_count: 1
```

```bash
cp ~/chief-of-staff/nudge-log.yaml ~/.claude/nudge-log.yaml
```

**Step 4: Commit**

```bash
cd ~/chief-of-staff
git add nudge-config.yaml nudge-log.yaml
git commit -m "feat: add nudge tracker configuration and log files"
```

---

### Task 6: Create the /nudge command

**Files:**
- Create: `~/chief-of-staff/commands/nudge.md`
- Copy to: `~/.claude/commands/nudge.md`

**Step 1: Write the nudge command**

Create `commands/nudge.md`:

```markdown
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
```

**Step 2: Copy to installed location**

```bash
cp ~/chief-of-staff/commands/nudge.md ~/.claude/commands/nudge.md
```

**Step 3: Test the nudge command**

Run `/nudge` in Claude Code. Expected behavior:
- Reads nudge-config.yaml for settings
- Scans Gmail and WhatsApp for stale threads
- Filters out no-reply addresses and closing messages
- Presents stale threads sorted by age
- After selection, drafts nudges using voice/ examples
- Shows draft-edit-send loop for each nudge
- Updates nudge-log.yaml after sending

**Step 4: Commit**

```bash
cd ~/chief-of-staff
git add commands/nudge.md
git commit -m "feat: add /nudge command for follow-up tracking"
```

---

### Task 7: Seed voice directory with example emails

**Files:**
- Create: 2-3 files in `~/chief-of-staff/voice/`

**Step 1: Gather example emails**

Find 2-3 sent emails that represent your natural writing style across different contexts:
- One casual/friendly reply
- One professional/formal response
- One follow-up or request

**Step 2: Create seed files**

For each email, create a YAML file in `voice/`. Example:

```yaml
# voice/seed-casual-reply.yaml
timestamp: 2026-03-04T00:00:00
context: "Casual reply to a colleague about a project update"
channel: gmail
recipient: "Example"
edited_version: |
  [Paste your actual sent email here]
tags: [casual, colleague, update]
```

Note: seed files have `edited_version` only (no `original_draft`) since these are pre-existing examples of your voice.

**Step 3: Commit**

```bash
cd ~/chief-of-staff
git add voice/
git commit -m "feat: seed voice directory with writing style examples"
```

---

### Task 8: End-to-end verification

**Step 1: Verify /gm works**

Run `/gm`. Confirm it produces a morning briefing with calendar, tasks, and inbox scan.

**Step 2: Verify /triage with draft-edit-send loop**

Run `/triage`. Confirm:
- Gmail and WhatsApp are both scanned
- Items are presented by tier
- Each item shows the `[s]end | [e]dit | [skip] | [r]egenerate | [g]mail draft` menu
- Editing a draft and sending creates a `.yaml` file in `~/.claude/voice/`
- Choosing `[g]mail draft` creates a draft visible in Gmail

**Step 3: Verify /nudge**

Run `/nudge`. Confirm:
- Config is loaded from nudge-config.yaml
- Stale threads are found and presented
- Nudge drafts use voice/ examples for tone
- Sending a nudge updates nudge-log.yaml
- Running `/nudge` again respects cooldown and max nudge limits

**Step 4: Verify voice learning improves over time**

After 5+ edits via the draft-edit-send loop:
- Check `~/.claude/voice/` for accumulated pairs
- Run `/triage` again and compare draft quality to earlier drafts
- Drafts should start matching your vocabulary and sentence structure

**Step 5: Final commit**

```bash
cd ~/chief-of-staff
git add -A
git commit -m "feat: complete communication chief of staff setup"
```

---

## Task Dependency Order

```
Task 1 (Fork + setup)
  └─▶ Task 2 (Voice directory)
  └─▶ Task 5 (Nudge config)
       │
       ▼
  Task 3 (CLAUDE.md voice instructions)
       │
       ▼
  Task 4 (Triage draft-edit-send loop)
  Task 6 (Nudge command)
       │
       ▼
  Task 7 (Seed voice examples)
       │
       ▼
  Task 8 (End-to-end verification)
```

Tasks 2 and 5 can run in parallel after Task 1.
Tasks 4 and 6 can run in parallel after Task 3.
Task 7 depends on Task 2.
Task 8 depends on everything.
