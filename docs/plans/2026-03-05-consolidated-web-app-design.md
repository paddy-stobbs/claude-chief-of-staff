# Chief of Staff: Consolidated Web App Design

**Date:** 2026-03-05
**Status:** Approved
**Author:** Paddy Stobbs + Claude

## Problem

The Chief of Staff system is split across three separate tools:
1. **CLI system** (chief-of-staff) — triage, nudges, tasks, contacts, morning briefing
2. **Todo app** — standalone Firebase PWA (vanilla JS, ~11K lines)
3. **Meeting notes app** — standalone Firebase + Cloud Functions pipeline

This creates three problems:
- **No visibility** — can't glance at tasks/meetings/nudges without running CLI commands
- **Interaction friction** — editing drafts in a terminal is clunky
- **Fragmentation** — three apps, three codebases, three mental models

## Solution

A single consolidated web app that replaces all three tools with a unified interface. The existing CLI system continues to run in parallel until the web app is proven.

## Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Frontend | React + Vite (SPA) | No SSR needed (app is fully authenticated). Simpler than Next.js. |
| Styling | Tailwind CSS + shadcn/ui | Already using Tailwind. shadcn/ui provides polished, ownable components. |
| Hosting | Firebase Hosting | Already using it for both existing apps. One platform. |
| Database | Firebase Realtime DB | Already using it in both existing apps. No migration needed. |
| Auth | Firebase Auth + Google OAuth | Already using it. Extend OAuth scopes for Gmail read + draft. |
| Backend | Firebase Cloud Functions | Already using it. Handles long-running Claude API calls (up to 9 min). |
| AI | Claude API (from Cloud Functions) | Already proven in meeting notes app. |

### Key decisions

- **No Next.js/Vercel.** SSR adds complexity for zero benefit behind auth. Staying on Firebase keeps one platform.
- **React over Svelte.** shadcn/ui ecosystem is mature and accelerates UI development. Claude generates excellent React code.
- **Firebase Realtime DB over Firestore.** Avoiding migration risk. Both existing apps use Realtime DB.
- **Cloud Functions over Vercel serverless.** No timeout issues on long Claude API calls. Already proven.

## Architecture

```
+---------------------------------------------+
|           React + Vite SPA                   |
|           (Firebase Hosting)                 |
|                                              |
|  /dashboard  /triage  /tasks  /nudges        |
|  /meetings                                   |
|                                              |
|           Firebase SDK (client)              |
+---------------------+------------------------+
                      |
              +-------+--------+
              |   Firebase      |
              |  Realtime DB    |
              |  Cloud Funcs    |
              |  Auth + OAuth   |
              +--+----+----+---+
                 |    |    |
           Gmail GCal Notion  Claude API
```

The frontend is a pure SPA. All AI processing, external API calls, and heavy computation happen in Cloud Functions. The frontend reads/writes Firebase Realtime DB and triggers jobs via the existing job queue pattern.

## Views / Routes

### `/` Dashboard
- Pre-computed data refreshed by scheduled Cloud Function (7am daily + on-demand)
- Shows: today's meetings, priority tasks (due today/overdue), due nudges, stale contacts (from Notion CRM)
- Raw structured data loads instantly from Firebase cache
- AI synthesis (e.g. priority recommendations) available on-demand

### `/triage` Email Triage
- Fetches inbox via Gmail API (Cloud Function)
- Classifies emails by tier: Tier 1 (now), Tier 2 (today), Tier 3 (FYI)
- For each email, Claude generates a draft reply
- User edits inline, saves (triggering voice learning), optionally pushes to Gmail draft
- Triage modes: quick (Tier 1 only), full (all tiers)

### `/tasks` Todo List
- Ported from existing todo app
- Business + Personal categories, Today/Later divider, drag-and-drop, notes editor
- Google Calendar integration (existing)
- NEW: "Delegate to Claude" button on each task
  - Claude reads the task description, determines what's needed
  - Has tool access to Gmail API, GCal, Notion
  - Runs as an agent loop in Cloud Functions
  - Returns output for user review (draft, research, action taken)
  - User reviews, edits, approves

### `/nudges` Follow-Up Tracker
- Scans Gmail for stale threads (configurable window, default 3 days)
- Cross-references with contact CRM data from Firebase
- Claude drafts nudge messages
- User edits inline, saves (voice learning), optionally pushes to Gmail draft
- Nudge guardrails: max 2 per contact per thread, 5-day cooldown

### `/meetings` Meeting Prep + Notes
- Shows today's meetings from GCal
- **Prep:** Click to generate prep notes. Cloud Function pulls context from:
  - Gmail (recent threads with attendees)
  - GCal (past meetings with attendees)
  - Contacts DB (CRM notes, relationship tier)
  - Meeting Notes DB (past meeting notes with attendees)
  - Claude synthesizes into actionable prep notes
- **Notes:** Ported from existing meeting notes app
  - Process Granola transcripts or paste raw transcripts
  - Meeting type selection: sales vs. interview (different note templates)
  - Edit notes inline, save (voice learning)
  - Upload to Notion (Team or Personal hub)
  - Sync edits back from Notion
- **Post-meeting:** Auto-suggest tasks and follow-up emails from meeting notes

### Contact Data (no dedicated view)
- Notion CRM remains the source of truth (team collaborates there)
- Contact context surfaces within other views:
  - Triage: shows contact tier, last interaction, CRM notes when replying
  - Nudges: cross-references CRM for relationship context
  - Meeting prep: pulls contact history and notes for attendees
  - Dashboard: surfaces stale contacts (Tier 1 >14d, Tier 2 >30d, Tier 3 >60d)
- Cloud Functions query Notion API for contact data as needed

## Core Component: Draft/Edit/Learn

Reusable component used across triage, nudges, meeting prep, meeting notes, and task delegation.

### Flow
1. Cloud Function generates draft (Claude API + voice pairs as few-shot examples)
2. Draft appears in inline rich text editor (editable)
3. User edits in place
4. User clicks **Save**:
   - If meaningful edits were made (>5 characters changed, not just typos):
     - Stores before/after pair in `voice_pairs/` in Firebase
     - Auto-tags with context: recipient, channel, topic, source (triage|nudge|meeting_prep|task)
   - Updates the draft in Firebase
5. User optionally clicks **Push to Gmail Draft**:
   - Creates a draft in Gmail via Gmail API (read + draft scopes)
   - User opens Gmail to review and send

### Voice Learning
- Voice pairs stored in Firebase (replacing local YAML files once system is proven)
- When generating drafts, Cloud Function queries top 3-5 most relevant pairs:
  - Same recipient > same topic/tags > same channel > same source type
- Pairs include timestamp for recency weighting

## Data Model (Firebase Realtime DB)

```
users/{uid}/
  tasks/
    business/[index] -> { text, completed, id, createdAt, updatedAt, notes?, schedule? }
    personal/[index] -> { text, completed, id, createdAt, updatedAt, notes?, schedule? }
    dividers/ -> { business: number, personal: number }
  archived/
    business/[index] -> { text, completed, archivedAt, ... }
    personal/[index] -> { text, completed, archivedAt, ... }
  meetings/{meetingId}
    title, created_at, attendees[], transcript, transcript_entries[], pushed_at, processed
  notes/{noteId}
    title, date, content, meeting_id, meeting_type, attendees[],
    notion_uploaded, notion_page_id, notion_hub, generated_at, updated_at
  transcripts/{noteId}
    content, updated_at
  triage/
    {sessionId}/
      emails[] -> { id, from, subject, snippet, tier, date }
      drafts[] -> { emailId, original_draft, current_draft, status (draft|saved|pushed) }
  nudges/
    {threadId}/ -> { thread_subject, contact, last_activity, draft, status, nudge_count, last_nudged }
  voice_pairs/{pairId}
    original_draft, edited_version, context: { recipient, channel, topic, tags[], source },
    created_at
  meeting_prep/{meetingId}
    prep_notes, generated_at, meeting_title, attendees[], sources_used[]
  dashboard_cache/
    last_refreshed, meetings_today[], priority_tasks[], due_nudges[], stale_contacts[]
  jobs/{jobId}
    type, status (pending|processing|complete|error), params{}, result{}, error?,
    created_at, updated_at
```

## Cloud Functions

| Function | Trigger | Description |
|----------|---------|-------------|
| `triageInbox` | Job queue | Fetch Gmail inbox, classify by tier, generate draft replies |
| `generateDraft` | Job queue | Generate a single draft with voice pair context |
| `generateNudges` | Job queue | Scan Gmail for stale threads, generate nudge drafts |
| `generateMeetingPrep` | Job queue | Pull cross-source context, generate prep notes |
| `generateMeetingNotes` | Job queue | Process transcript into structured notes (existing) |
| `delegateTask` | Job queue | Agent loop: read task, use tools (Gmail/GCal/Notion), return output |
| `pushToGmailDraft` | HTTP callable | Create a draft in Gmail via API |
| `refreshDashboard` | Scheduled (7am) + HTTP callable | Pre-compute dashboard data |
| `uploadToNotion` | Job queue (existing) | Push notes to Notion Document Hub |
| `syncFromNotion` | Job queue (existing) | Pull edits back from Notion |
| `autoCreateTasks` | Job queue | Scan meeting notes for action items, suggest tasks |

### Job Queue Pattern (existing)
All AI-heavy operations use the same pattern:
1. Frontend writes a job to `jobs/{jobId}` with `status: "pending"`
2. Cloud Function triggers on job creation/update
3. Function processes the job, updates `status` to `"processing"` then `"complete"` or `"error"`
4. Frontend listens for job status changes in real-time

## Auth & OAuth

- Firebase Auth with Google Sign-In (existing)
- Google OAuth scopes:
  - `gmail.readonly` — read inbox for triage
  - `gmail.compose` — create drafts
  - `calendar.readonly` — read events for meetings/prep
- OAuth tokens stored securely via Firebase Auth (Google provider stores refresh tokens)
- Notion API token: stored in Cloud Functions secrets (existing)
- Claude API key: stored in Cloud Functions secrets (existing)

## Gmail Integration

**Read-only + drafts approach** (not full send):
- App reads emails and generates draft replies
- Pushes approved drafts to Gmail as drafts
- User opens Gmail to review and send
- Avoids the `gmail.send` scope and "app sent on your behalf" risk
- Aligns with existing CLAUDE.md guardrail: "never send without explicit approval"

## Migration Strategy

Each phase is independently useful. The existing CLI system and standalone apps continue running in parallel throughout.

### Phase 0: Scaffold
- Create new React + Vite project
- Firebase Auth integration
- Basic routing (react-router)
- shadcn/ui setup
- Deploy empty shell to Firebase Hosting
- **Exit criteria:** Authenticated app with navigation between empty views

### Phase 1: Tasks
- Port todo app to React components
- Business/Personal categories, Today/Later divider, drag-and-drop
- Notes editor
- GCal integration
- Read/write from existing Firebase Realtime DB structure
- **Exit criteria:** Feature parity with existing todo app

### Phase 2: Meeting Notes
- Port meeting notes app to React components
- Transcript processing (Granola + manual paste)
- Note generation via existing Cloud Functions
- Notion upload + sync
- Add draft/edit/learn component (first use)
- **Exit criteria:** Feature parity with existing meeting notes app, plus voice learning

### Phase 3: Triage + Nudges
- Google OAuth for Gmail (read + draft scopes)
- Triage view: inbox, tier classification, draft generation
- Nudge view: stale thread detection, draft generation
- Draft/edit/learn component reused from Phase 2
- Push to Gmail draft
- **Exit criteria:** Full triage and nudge flows working in web app

### Phase 4: Meeting Prep
- New Cloud Function: cross-source context gathering + Claude synthesis
- Meeting prep view integrated into /meetings
- Edit/learn on prep notes
- **Exit criteria:** Meeting prep generating useful notes from GCal + Gmail + Notion + past meetings

### Phase 5: Dashboard + Delegate
- Scheduled Cloud Function for dashboard pre-computation
- Dashboard view with today's meetings, priority tasks, due nudges
- "Delegate to Claude" on tasks: agent loop Cloud Function with tool access
- **Exit criteria:** Dashboard loads instantly, task delegation produces useful output

### Phase 6: Polish + Retire
- Auto-task creation from meeting notes
- Chat interface for querying across all context (stretch)
- Confidence-test against CLI system
- Retire standalone todo app, meeting notes app, and CLI commands one by one
- **Exit criteria:** All functionality consolidated, old apps retired

## Non-Goals (for now)
- WhatsApp triage in the web app (keep in CLI; revisit with WhatsApp Business API later)
- Direct email sending (Gmail drafts only)
- Mobile-native app (PWA is sufficient)
- Multi-user / team features
- Public-facing anything (this is a personal tool)

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Gmail OAuth scopes rejected by Google | Stay in "testing" mode (up to 100 users). Personal tool doesn't need verification. |
| Cloud Function cold starts slow down UX | Use min-instances for critical functions. Job queue pattern means UI isn't blocking. |
| Voice learning data quality | Only save pairs with meaningful edits (>5 char diff). Weight by recency. |
| Migration breaks existing apps | Parallel running throughout. No data migration — same Firebase DB. |
| Scope creep | Phased approach. Each phase is independently useful. Stop anytime. |
