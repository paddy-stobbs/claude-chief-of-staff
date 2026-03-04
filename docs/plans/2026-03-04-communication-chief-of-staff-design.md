# Communication Chief of Staff - Design Document

**Date:** 2026-03-04
**Status:** Approved
**Base:** Fork of [claude-chief-of-staff](https://github.com/mimurchison/claude-chief-of-staff)

## Problem

Managing Gmail and WhatsApp communication is time-consuming. Need a system that triages messages, drafts responses that sound like me, learns my writing style over time from edits I make, and follows up on threads where I'm waiting for a reply.

## Architecture Overview

Fork claude-chief-of-staff and extend it with three new capabilities: voice learning, nudge tracking, and a draft-edit-send loop.

```
┌─────────────────────────────────────────────────┐
│              Claude Code CLI                     │
│  /triage  /draft  /nudge  /gm  /voice-review    │
└──────────┬──────────────────────┬───────────────┘
           │                      │
     ┌─────▼─────┐         ┌─────▼──────┐
     │  Existing  │         │  New: Your  │
     │  CCoS Core │         │  Extensions │
     │            │         │             │
     │  Triage    │         │  Voice      │
     │  Contacts  │         │  Learning   │
     │  Goals     │         │  Nudge      │
     │  Briefing  │         │  Tracker    │
     └─────┬──────┘         └──────┬──────┘
           │                       │
     ┌─────▼───────────────────────▼──────┐
     │         MCP Server Layer            │
     │                                     │
     │  Gmail MCP    WhatsApp MCP          │
     │  Calendar MCP (optional)            │
     └─────────────────────────────────────┘
```

### What we keep from chief-of-staff (unchanged)

- `/gm` morning briefing
- `/my-tasks` task management
- `/enrich` contact enrichment
- `goals.yaml` goal alignment
- `my-tasks.yaml` task tracking
- `schedules.yaml` automation
- Gmail MCP, Google Calendar MCP, Slack MCP (optional)
- Contact profiles

### What we modify

- `/triage` -- extend with draft-edit-send loop and WhatsApp parity
- `CLAUDE.md` -- add voice learning instructions so system pulls from `voice/` when drafting
- WhatsApp MCP -- elevate from optional to required

### What we add

- Voice learning system (`voice/` directory)
- Nudge tracker (`/nudge` command)
- Draft-edit-send loop (integrated into `/triage` and `/nudge`)

## Component 1: Voice Learning System

**Goal:** After ~20-30 edited drafts, the system writes emails that sound like the user.

**Mechanism:** Few-shot prompting with draft-vs-edit pairs. No fine-tuning or model training.

### Storage

A `voice/` directory in the project root. One YAML file per captured edit:

```yaml
# voice/2026-03-04-reply-to-john-fundraising.yaml
timestamp: 2026-03-04T09:15:00
context: "Reply to investor asking about Series B timeline"
channel: gmail
original_draft: |
  Hi John,
  Thank you for reaching out about our Series B timeline...
edited_version: |
  John - appreciate you asking. We're targeting Q3 but
  keeping it flexible depending on how pipeline shapes up...
tags: [investor, fundraising, casual-tone]
```

### Example selection

When drafting a response, the system:
1. Scans `voice/` for pairs with similar context (same contact, same topic, same channel)
2. Pulls the 3-5 most relevant pairs into the prompt as few-shot examples
3. At < 30 emails/day with flat files, this is a simple directory scan -- no database needed

### Edit capture

- System generates a draft during `/triage` or `/nudge`
- User edits the draft in the CLI
- On send, system diffs original draft vs edited version
- If there's a meaningful diff (not just typo fixes), saves the pair to `voice/`
- If sent without edits, no pair is saved (draft was good, no correction needed)

### Cold start / bootstrapping

- First ~10-20 drafts won't have many examples
- User can optionally seed `voice/` with sent emails they're proud of (as `edited_version` with no `original_draft`)
- `CLAUDE.md` personality section carries the load until enough pairs accumulate

## Component 2: Nudge Tracker

**Goal:** Surface threads where the user sent the last message and hasn't heard back, then draft follow-ups.

### Flow

1. `/nudge` queries Gmail + WhatsApp for threads where user sent the last message > N days ago
2. Filters out newsletters, no-reply addresses, "thanks"/"got it" closers, and recently nudged threads
3. Presents stale threads ranked by age
4. User selects which to nudge
5. System drafts nudges using `voice/` examples for tone matching
6. User reviews, edits, sends via the standard draft-edit-send loop

### Configuration (`nudge-config.yaml`)

```yaml
default_window_days: 3
channels:
  gmail: true
  whatsapp: true
ignore_patterns:
  - "noreply@"
  - "no-reply@"
  - "notifications@"
max_nudges_per_contact: 2
nudge_cooldown_days: 5
```

### State tracking (`nudge-log.yaml`)

Records which threads have been nudged and when. Prevents over-nudging. Checked each time `/nudge` runs.

## Component 3: Draft-Edit-Send Loop

**Goal:** Unified workflow for reviewing, editing, and sending responses across Gmail and WhatsApp.

### The loop

```
/triage or /nudge
  │
  ▼
1. System presents prioritized items
   [RESPOND NOW]
   1. John - Series B timeline question
   2. Sarah - Contract review request
   [HANDLE TODAY]
   3. Dev team - API access follow-up
  │
  ▼
2. User selects an item (or "draft all")
  │
  ▼
3. System generates draft
   - Pulls relevant voice/ examples
   - Uses CLAUDE.md personality
   - Incorporates thread context
  │
  ▼
4. User reviews:
   [s]end  [e]dit  [skip]  [r]egenerate  [g]mail draft
  │
  ├─ send → message sent via MCP, no voice pair saved
  ├─ edit → user modifies, then send → voice pair saved
  ├─ skip → move to next item
  ├─ regenerate → new draft with different approach
  └─ gmail draft → saved as draft in Gmail via MCP,
                    user finishes in Gmail web app
                    (for attachments, formatting review)
                    voice pair still captured from CLI edits
  │
  ▼
5. Loop to next item
```

### "Draft all" mode

User says "draft all" after triage. System generates drafts for every actionable item, then user walks through them one by one.

### Channel parity

Same review loop for Gmail and WhatsApp. Only difference is which MCP server handles the send. The `[g]mail draft` option is Gmail-only (WhatsApp has no drafts concept).

### Gmail web app handoff

When user chooses `[g]mail draft`, the system saves the draft via Gmail MCP's `create_draft` endpoint. User opens Gmail to review formatting, add attachments, and send. Any further edits made in the Gmail web app are not captured in `voice/` -- the CLI-edited version is the style signal.

## File Structure

```
claude-chief-of-staff/          (forked repo)
│
├── CLAUDE.md                   (existing - extend with voice learning instructions)
├── goals.yaml                  (existing - unchanged)
├── my-tasks.yaml               (existing - unchanged)
├── schedules.yaml              (existing - unchanged)
│
├── commands/
│   ├── gm.md                   (existing - unchanged)
│   ├── triage.md               (existing - extend for draft loop + WhatsApp)
│   ├── my-tasks.md             (existing - unchanged)
│   ├── enrich.md               (existing - unchanged)
│   └── nudge.md                (NEW)
│
├── contacts/                   (existing - unchanged)
│
├── voice/                      (NEW)
│   ├── README.md
│   └── *.yaml                  (draft-vs-edit pairs, accumulate over time)
│
├── nudge-config.yaml           (NEW)
├── nudge-log.yaml              (NEW)
│
└── docs/
    └── plans/
        └── 2026-03-04-communication-chief-of-staff-design.md
```

### New files to create

- `commands/nudge.md` -- nudge command definition
- `voice/README.md` -- explains the voice pair format
- `nudge-config.yaml` -- nudge configuration
- `nudge-log.yaml` -- nudge state tracking

### Files to modify

- `commands/triage.md` -- add draft-edit-send loop with `[g]mail draft` option, WhatsApp parity
- `CLAUDE.md` -- add voice learning instructions

## Decisions made

- **Fork chief-of-staff** over building from scratch (saves weeks of MCP wiring)
- **Style matching** is the learning goal (not decision pattern learning)
- **Flat-file storage** for voice pairs (sufficient at < 30 emails/day)
- **Track all sent emails** for nudge candidates, filter at review time
- **Few-shot prompting** over fine-tuning (simpler, no training pipeline needed)
- **Gmail draft handoff** for emails needing attachments/formatting review
