# Chief of Staff Web App — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a consolidated React web app that merges the CLI chief-of-staff, todo app, and meeting notes app into a single interface with 5 views: Dashboard, Triage, Tasks, Nudges, Meetings.

**Architecture:** React + Vite SPA deployed to Firebase Hosting. Firebase Realtime DB for data, Cloud Functions for AI/API work (Claude API, Gmail API, Notion API, GCal API). Job queue pattern for all async operations. Single reusable Draft/Edit/Learn component for voice learning across views.

**Tech Stack:** React 19, Vite, TypeScript, Tailwind CSS, shadcn/ui, Firebase (Hosting + Realtime DB + Cloud Functions + Auth), Anthropic SDK, Google APIs (Gmail, Calendar), Notion API.

**Design doc:** `docs/plans/2026-03-05-consolidated-web-app-design.md`

**Existing codebases to reference:**
- Todo app: `/Users/patrickstobbs/Documents/coding/personal/todo-app/` (vanilla JS, Firebase project: `paddy-s-todo-list`)
- Meeting notes: `/Users/patrickstobbs/Documents/coding/personal/meeting_notes/` (vanilla JS + Cloud Functions, Firebase project: `meeting-notes-paddy`)
- CLI chief-of-staff: `/Users/patrickstobbs/Documents/coding/personal/chief-of-staff/`

**IMPORTANT: Firebase project decision.** The two existing apps use different Firebase projects (`paddy-s-todo-list` and `meeting-notes-paddy`). The consolidated app should use a NEW Firebase project so both old apps keep running untouched. Data migration happens per-phase when ready.

---

## Phase 0: Scaffold

**Exit criteria:** Authenticated app with sidebar navigation between 5 empty views, deployed to Firebase Hosting.

### Task 0.1: Create project and initialize React + Vite + TypeScript

**Files:**
- Create: `/Users/patrickstobbs/Documents/coding/personal/chief-of-staff-app/`

**Step 1: Scaffold Vite project**

```bash
cd /Users/patrickstobbs/Documents/coding/personal
npm create vite@latest chief-of-staff-app -- --template react-ts
cd chief-of-staff-app
npm install
```

**Step 2: Verify it runs**

```bash
npm run dev
```

Expected: Vite dev server starts at localhost:5173, shows default React page.

**Step 3: Initialize git**

```bash
git init
git add .
git commit -m "feat: scaffold React + Vite + TypeScript project"
```

---

### Task 0.2: Install and configure Tailwind CSS

**Files:**
- Modify: `chief-of-staff-app/package.json`
- Modify: `chief-of-staff-app/src/index.css`
- Create: `chief-of-staff-app/tailwind.config.js`
- Create: `chief-of-staff-app/postcss.config.js`

**Step 1: Install Tailwind**

```bash
cd /Users/patrickstobbs/Documents/coding/personal/chief-of-staff-app
npm install -D tailwindcss @tailwindcss/vite
```

**Step 2: Configure Vite plugin**

In `vite.config.ts`:
```typescript
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
})
```

**Step 3: Add Tailwind import to CSS**

Replace contents of `src/index.css`:
```css
@import "tailwindcss";
```

**Step 4: Verify Tailwind works**

Add a Tailwind class to `App.tsx` (e.g., `<h1 className="text-3xl font-bold text-blue-600">Test</h1>`), verify it renders styled in browser.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add Tailwind CSS"
```

---

### Task 0.3: Install and configure shadcn/ui

**Files:**
- Create: `chief-of-staff-app/components.json`
- Create: `chief-of-staff-app/src/components/ui/` (generated)
- Modify: `chief-of-staff-app/src/lib/utils.ts`

**Step 1: Initialize shadcn**

```bash
npx shadcn@latest init
```

Choose: TypeScript, default style, default color, CSS variables: yes.

**Step 2: Add initial components we'll need**

```bash
npx shadcn@latest add button card tabs textarea badge separator skeleton toast dropdown-menu dialog scroll-area
```

**Step 3: Verify a component renders**

Import and render a `<Button>` in `App.tsx`. Verify it shows styled in browser.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add shadcn/ui components"
```

---

### Task 0.4: Create new Firebase project and configure

**Files:**
- Create: `chief-of-staff-app/firebase.json`
- Create: `chief-of-staff-app/.firebaserc`
- Create: `chief-of-staff-app/database.rules.json`
- Create: `chief-of-staff-app/src/lib/firebase.ts`

**Step 1: Create Firebase project**

Go to Firebase Console (https://console.firebase.google.com), create project named `chief-of-staff-paddy`. Enable:
- Authentication (Google provider)
- Realtime Database (region: europe-west1)
- Hosting
- Cloud Functions

**Step 2: Install Firebase CLI tools and SDK**

```bash
npm install firebase
firebase init
```

Select: Hosting (dist directory), Database, Functions. Set `dist` as public directory. Configure as SPA (rewrite all to index.html).

**Step 3: Create Firebase client config**

Create `src/lib/firebase.ts`:
```typescript
import { initializeApp } from 'firebase/app'
import { getAuth, GoogleAuthProvider } from 'firebase/auth'
import { getDatabase } from 'firebase/database'

const firebaseConfig = {
  // Paste config from Firebase Console
  apiKey: "...",
  authDomain: "chief-of-staff-paddy.firebaseapp.com",
  databaseURL: "https://chief-of-staff-paddy-default-rtdb.europe-west1.firebasedatabase.app",
  projectId: "chief-of-staff-paddy",
  storageBucket: "chief-of-staff-paddy.firebasestorage.app",
  messagingSenderId: "...",
  appId: "..."
}

const app = initializeApp(firebaseConfig)
export const auth = getAuth(app)
export const db = getDatabase(app)
export const googleProvider = new GoogleAuthProvider()
```

**Step 4: Set up initial database rules**

Create `database.rules.json`:
```json
{
  "rules": {
    "users": {
      "$uid": {
        ".read": "$uid === auth.uid",
        ".write": "$uid === auth.uid"
      }
    },
    ".read": false,
    ".write": false
  }
}
```

**Step 5: Commit**

```bash
git add .
git commit -m "feat: configure Firebase project"
```

---

### Task 0.5: Implement Firebase Auth with Google Sign-In

**Files:**
- Create: `chief-of-staff-app/src/hooks/useAuth.ts`
- Create: `chief-of-staff-app/src/components/LoginPage.tsx`
- Modify: `chief-of-staff-app/src/App.tsx`

**Step 1: Create auth hook**

Create `src/hooks/useAuth.ts`:
```typescript
import { useState, useEffect } from 'react'
import { onAuthStateChanged, signInWithPopup, signOut as firebaseSignOut, User } from 'firebase/auth'
import { auth, googleProvider } from '@/lib/firebase'

export function useAuth() {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (user) => {
      setUser(user)
      setLoading(false)
    })
    return unsubscribe
  }, [])

  const signIn = () => signInWithPopup(auth, googleProvider)
  const signOut = () => firebaseSignOut(auth)

  return { user, loading, signIn, signOut }
}
```

**Step 2: Create login page**

Create `src/components/LoginPage.tsx` — simple centered card with Google Sign-In button.

**Step 3: Wire up in App.tsx**

`App.tsx` should:
- Show loading spinner while auth state resolves
- Show `LoginPage` if no user
- Show main app layout if authenticated

**Step 4: Test sign-in flow**

Run dev server, click Sign In, verify Google popup works and user state persists on refresh.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add Firebase Auth with Google Sign-In"
```

---

### Task 0.6: Set up routing and app shell with sidebar

**Files:**
- Create: `chief-of-staff-app/src/components/layout/AppShell.tsx`
- Create: `chief-of-staff-app/src/components/layout/Sidebar.tsx`
- Create: `chief-of-staff-app/src/pages/Dashboard.tsx`
- Create: `chief-of-staff-app/src/pages/Triage.tsx`
- Create: `chief-of-staff-app/src/pages/Tasks.tsx`
- Create: `chief-of-staff-app/src/pages/Nudges.tsx`
- Create: `chief-of-staff-app/src/pages/Meetings.tsx`
- Modify: `chief-of-staff-app/src/App.tsx`

**Step 1: Install react-router**

```bash
npm install react-router-dom
```

**Step 2: Create page stubs**

Create each page file (`Dashboard.tsx`, `Triage.tsx`, `Tasks.tsx`, `Nudges.tsx`, `Meetings.tsx`) as simple components with the page name as heading. E.g.:

```typescript
export default function Dashboard() {
  return <div className="p-6"><h1 className="text-2xl font-bold">Dashboard</h1></div>
}
```

**Step 3: Create Sidebar**

Create `src/components/layout/Sidebar.tsx`:
- 5 nav items: Dashboard, Triage, Tasks, Nudges, Meetings
- Use `NavLink` from react-router for active state styling
- Icons for each item (use lucide-react, installed with shadcn)
- User info + sign out at bottom

**Step 4: Create AppShell**

Create `src/components/layout/AppShell.tsx`:
- Fixed sidebar on left (w-64)
- Main content area with `<Outlet />` from react-router
- Responsive: collapse sidebar on mobile (stretch — can do later)

**Step 5: Wire up routing in App.tsx**

```typescript
import { BrowserRouter, Routes, Route } from 'react-router-dom'

// After auth check:
<BrowserRouter>
  <Routes>
    <Route element={<AppShell />}>
      <Route path="/" element={<Dashboard />} />
      <Route path="/triage" element={<Triage />} />
      <Route path="/tasks" element={<Tasks />} />
      <Route path="/nudges" element={<Nudges />} />
      <Route path="/meetings" element={<Meetings />} />
    </Route>
  </Routes>
</BrowserRouter>
```

**Step 6: Verify navigation**

Run dev server. Click each sidebar item. Verify URL changes and correct page renders. Verify active state highlights on sidebar.

**Step 7: Commit**

```bash
git add .
git commit -m "feat: add routing and app shell with sidebar navigation"
```

---

### Task 0.7: Deploy to Firebase Hosting

**Files:**
- Modify: `chief-of-staff-app/firebase.json`
- Modify: `chief-of-staff-app/vite.config.ts`

**Step 1: Build**

```bash
npm run build
```

**Step 2: Verify firebase.json points to dist**

```json
{
  "hosting": {
    "public": "dist",
    "rewrites": [{ "source": "**", "destination": "/index.html" }]
  }
}
```

**Step 3: Deploy**

```bash
firebase deploy --only hosting
```

**Step 4: Verify**

Open the deployed URL. Sign in. Navigate between pages. All should work.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: deploy to Firebase Hosting"
```

---

### Task 0.8: Set up Cloud Functions directory

**Files:**
- Create: `chief-of-staff-app/functions/package.json`
- Create: `chief-of-staff-app/functions/index.js`

**Step 1: Initialize Cloud Functions**

If not already done during `firebase init`:

```bash
cd /Users/patrickstobbs/Documents/coding/personal/chief-of-staff-app
firebase init functions
```

Choose JavaScript, Node 22.

**Step 2: Install dependencies**

Match the existing meeting notes pattern:

```bash
cd functions
npm install @anthropic-ai/sdk firebase-admin firebase-functions
```

**Step 3: Create empty index.js**

```javascript
// Cloud Functions entry point
// Functions will be added per phase
```

**Step 4: Set up secrets**

```bash
firebase functions:secrets:set ANTHROPIC_API_KEY
firebase functions:secrets:set NOTION_API_KEY
```

**Step 5: Deploy functions**

```bash
cd ..
firebase deploy --only functions
```

**Step 6: Commit**

```bash
git add .
git commit -m "feat: set up Cloud Functions directory with secrets"
```

---

## Phase 1: Tasks

**Exit criteria:** Feature parity with existing todo app — Business/Personal categories, Today/Later divider, drag-and-drop, notes editor, GCal integration. Reading/writing Firebase Realtime DB.

**Reference:** Existing todo app at `/Users/patrickstobbs/Documents/coding/personal/todo-app/index.html`

**Data structure (from existing app):**
```
users/{uid}/
  tasks/
    business/[index] -> { text, completed, id, createdAt, updatedAt, notes?, schedule? }
    personal/[index] -> { text, completed, id, createdAt, updatedAt, notes?, schedule? }
    dividers/ -> { business: number, personal: number }
  archived/
    business/[index] -> { ... }
    personal/[index] -> { ... }
```

### Task 1.1: Create Firebase hooks for task data

**Files:**
- Create: `src/hooks/useFirebase.ts`
- Create: `src/hooks/useTasks.ts`
- Create: `src/types/task.ts`

**Step 1: Define task types**

Create `src/types/task.ts`:
```typescript
export interface TaskSchedule {
  date: string
  startTime: string
  endTime: string
  calendarEventId?: string
  calendarId?: string
  syncedAt?: number
  calendarSyncPending?: boolean
}

export interface Task {
  text: string
  completed: boolean
  id: string
  createdAt?: number
  updatedAt: number
  notes?: string
  schedule?: TaskSchedule
}

export type Category = 'business' | 'personal'

export interface TaskData {
  business: Task[]
  personal: Task[]
  dividers: { business: number; personal: number }
}
```

**Step 2: Create generic Firebase hook**

Create `src/hooks/useFirebase.ts` — generic hook for listening to a Realtime DB path with `onValue`.

**Step 3: Create tasks hook**

Create `src/hooks/useTasks.ts` — wraps `useFirebase` for the tasks path. Provides:
- `tasks: TaskData` (reactive)
- `addTask(category, text)`
- `updateTask(category, index, updates)`
- `deleteTask(category, index)`
- `reorderTask(category, fromIndex, toIndex)`
- `moveDivider(category, newPosition)`
- `archiveCompleted(category)`

Write to Firebase on every mutation (debounced 500ms, matching existing pattern).

**Step 4: Verify with a simple test**

In the Tasks page, display `JSON.stringify(tasks)`. Add a task via the Firebase console. Verify it appears in real-time.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add Firebase hooks for task CRUD"
```

---

### Task 1.2: Build task list component

**Files:**
- Create: `src/components/tasks/TaskList.tsx`
- Create: `src/components/tasks/TaskItem.tsx`
- Create: `src/components/tasks/TaskInput.tsx`
- Create: `src/components/tasks/DividerRow.tsx`
- Modify: `src/pages/Tasks.tsx`

**Step 1: Build TaskItem component**

`TaskItem.tsx` — single task row with:
- Checkbox (toggle completed)
- Text (click to edit inline)
- Drag handle
- Delete button (on hover)
- Visual difference for completed tasks (strikethrough, muted)

**Step 2: Build TaskInput component**

`TaskInput.tsx` — input field at top of each category for adding new tasks. Submit on Enter.

**Step 3: Build DividerRow component**

`DividerRow.tsx` — visual divider showing "Today" above / "Later" below. Text label, horizontal line.

**Step 4: Build TaskList component**

`TaskList.tsx` — renders a single category (business or personal):
- TaskInput at top
- Tasks above divider (Today)
- DividerRow
- Tasks below divider (Later)
- Archive button at bottom

**Step 5: Wire up Tasks page**

`Tasks.tsx` — two TaskList components side by side (or tabbed on mobile):
- Business (left)
- Personal (right)

**Step 6: Verify CRUD**

Add tasks, check/uncheck, delete. Verify changes persist on page reload (Firebase).

**Step 7: Commit**

```bash
git add .
git commit -m "feat: build task list with CRUD operations"
```

---

### Task 1.3: Add drag-and-drop for task reordering

**Files:**
- Modify: `src/components/tasks/TaskItem.tsx`
- Modify: `src/components/tasks/TaskList.tsx`
- Create: `src/hooks/useDragDrop.ts`

**Step 1: Create drag-and-drop hook**

Create `src/hooks/useDragDrop.ts` — encapsulates HTML5 drag API logic:
- Track dragged item index and category
- Calculate drop target position
- Adjust divider position when tasks cross it (matching existing logic from todo app lines 8722-8785)
- Return drag event handlers for TaskItem

Reference the existing drag logic in `/Users/patrickstobbs/Documents/coding/personal/todo-app/index.html` lines 8722-8785 for the divider adjustment algorithm.

**Step 2: Add drag handlers to TaskItem**

Make TaskItem draggable. Add drag handle icon. Apply drag-over styling.

**Step 3: Handle drop in TaskList**

On drop: call `reorderTask()` from useTasks hook. Re-render list.

**Step 4: Test drag-and-drop**

Drag tasks within a category. Drag tasks across the Today/Later divider. Verify divider position adjusts correctly. Verify order persists in Firebase.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add drag-and-drop task reordering"
```

---

### Task 1.4: Build task notes editor

**Files:**
- Create: `src/components/tasks/TaskNotes.tsx`
- Modify: `src/components/tasks/TaskItem.tsx`

**Step 1: Build TaskNotes component**

`TaskNotes.tsx` — expandable notes panel below a task:
- Rich text area supporting checkboxes (`[ ] text`), bullets (`- text`), and plain text
- Max 10,000 characters (matching existing validation)
- Auto-saves on change (debounced)
- Slash command menu (`/`) for inserting checkboxes, bullets (stretch — can simplify to plain textarea first)

Reference existing notes editor in todo app lines 5200-5800 for the serialization format.

**Step 2: Toggle notes visibility from TaskItem**

Click task text or a notes icon to expand/collapse the notes panel.

**Step 3: Test notes**

Add notes to a task. Reload page. Verify notes persist. Test checkboxes and bullets.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add task notes editor"
```

---

### Task 1.5: Add Google Calendar integration

**Files:**
- Create: `src/hooks/useGoogleCalendar.ts`
- Create: `src/components/tasks/TaskScheduler.tsx`
- Create: `src/components/tasks/CalendarView.tsx`
- Modify: `src/lib/firebase.ts`

**Note:** This requires setting up Google OAuth with calendar scopes. The existing todo app uses OAuth Client ID `1074858661938-5sq8qa4301afe5ibaq30eau50bgau2g7.apps.googleusercontent.com`. For the new Firebase project, you'll need to create a new OAuth client ID in Google Cloud Console for the new project.

**Step 1: Set up Google OAuth**

In Google Cloud Console for the new Firebase project:
- Enable Google Calendar API
- Create OAuth 2.0 Client ID (Web application)
- Add authorized origins for localhost:5173 and the Firebase Hosting URL
- Add scopes: `calendar.readonly`, `calendar.events`

Update `src/lib/firebase.ts`:
```typescript
googleProvider.addScope('https://www.googleapis.com/auth/calendar.readonly')
googleProvider.addScope('https://www.googleapis.com/auth/calendar.events')
```

**Step 2: Create GCal hook**

Create `src/hooks/useGoogleCalendar.ts`:
- `connectCalendar()` — trigger Google OAuth popup with calendar scopes
- `fetchEvents(startDate, endDate)` — GET events from GCal API
- `createEvent(task)` — POST event to GCal
- `updateEvent(eventId, updates)` — PATCH event
- `deleteEvent(eventId)` — DELETE event
- Token management: store in sessionStorage, refresh silently via GIS

Reference existing GCal integration in todo app lines 3628-3788 (OAuth flow) and 9668-9741 (fetch events), 4169-4265 (create/update/delete events).

**Step 3: Build TaskScheduler component**

`TaskScheduler.tsx` — modal or popover for scheduling a task:
- Date picker
- Start time / end time
- Creates GCal event and stores `schedule` object on task

**Step 4: Build CalendarView component**

`CalendarView.tsx` — optional right panel showing today's calendar:
- Time slots (7am-7pm)
- Events rendered as blocks
- Click to create task
- Drag to reschedule (stretch)

**Step 5: Test full flow**

Schedule a task. Verify event appears in Google Calendar. Edit time. Verify GCal updates. Delete schedule. Verify event removed.

**Step 6: Commit**

```bash
git add .
git commit -m "feat: add Google Calendar integration for tasks"
```

---

### Task 1.6: Update database rules for tasks

**Files:**
- Modify: `chief-of-staff-app/database.rules.json`

**Step 1: Add task validation rules**

Port the rules from the existing todo app (`/Users/patrickstobbs/Documents/coding/personal/todo-app/database.rules.json`). Add validation for tasks, dividers, and archived data.

**Step 2: Deploy rules**

```bash
firebase deploy --only database
```

**Step 3: Test**

Verify tasks still read/write correctly. Try writing invalid data from the Firebase console — should be rejected.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add database validation rules for tasks"
```

---

### Task 1.7: Data migration — copy tasks from old Firebase project

**Note:** This is optional during development. The user can continue using the old todo app until ready to switch. When ready:

**Step 1: Export data from old project**

```bash
firebase --project paddy-s-todo-list database:get /users > /tmp/todo-data.json
```

**Step 2: Import into new project**

```bash
firebase --project chief-of-staff-paddy database:set /users --data /tmp/todo-data.json
```

**Step 3: Verify**

Open the new app. Verify all tasks appear with correct categories, notes, and schedules.

**Step 4: Commit (rules/config only)**

No code changes — just a data operation.

---

## Phase 2: Meeting Notes

**Exit criteria:** Feature parity with existing meeting notes app — transcript processing, note generation, Notion upload/sync. Plus the Draft/Edit/Learn component (first use).

**Reference:** Existing meeting notes app at `/Users/patrickstobbs/Documents/coding/personal/meeting_notes/`

### Task 2.1: Port Cloud Functions from meeting notes app

**Files:**
- Copy to: `functions/generateNotes.js`
- Copy to: `functions/uploadToNotion.js`
- Copy to: `functions/syncFromNotion.js`
- Copy to: `functions/notionHelpers.js`
- Modify: `functions/index.js`
- Modify: `functions/package.json`

**Step 1: Copy Cloud Functions**

Copy the 4 files from `/Users/patrickstobbs/Documents/coding/personal/meeting_notes/functions/` into the new project's `functions/` directory. These are:
- `generateNotes.js` — transcript → notes via Claude API
- `uploadToNotion.js` — notes → Notion
- `syncFromNotion.js` — Notion → notes
- `notionHelpers.js` — markdown ↔ Notion blocks conversion

**Step 2: Update index.js to export functions**

```javascript
const { generateNotes } = require('./generateNotes')
const { uploadToNotion } = require('./uploadToNotion')
const { syncFromNotion } = require('./syncFromNotion')

exports.generateNotes = generateNotes
exports.uploadToNotion = uploadToNotion
exports.syncFromNotion = syncFromNotion
```

**Step 3: Set secrets on new project**

```bash
firebase functions:secrets:set ANTHROPIC_API_KEY
firebase functions:secrets:set NOTION_API_KEY
```

**Step 4: Deploy and test**

```bash
firebase deploy --only functions
```

Create a test job in the Firebase console under `users/{uid}/jobs/` with `type: "generate"`, `status: "pending"`, and a test `meeting_id`. Verify the function triggers and processes.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: port Cloud Functions from meeting notes app"
```

---

### Task 2.2: Create job queue hook

**Files:**
- Create: `src/hooks/useJobs.ts`

**Step 1: Build the hook**

Create `src/hooks/useJobs.ts` — mirrors the `startJob()` / `watchJob()` pattern from the existing meeting notes app (`firebase-client.js` lines 254-304):

```typescript
export function useJobs() {
  const { user } = useAuth()

  const startJob = async (params: {
    type: string
    meeting_id?: string
    meeting_type?: string
    [key: string]: any
  }) => {
    const jobRef = push(ref(db, `users/${user.uid}/jobs`))
    await set(jobRef, {
      ...params,
      status: 'pending',
      created_at: new Date().toISOString(),
      updated_at: new Date().toISOString(),
    })
    return watchJob(jobRef.key!)
  }

  const watchJob = (jobId: string): Promise<any> => {
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => reject(new Error('Job timed out')), 120000)
      const jobRef = ref(db, `users/${user.uid}/jobs/${jobId}`)
      const unsubscribe = onValue(jobRef, (snap) => {
        const job = snap.val()
        if (job?.status === 'complete') {
          clearTimeout(timeout)
          unsubscribe()
          resolve(job.result || {})
        } else if (job?.status === 'error') {
          clearTimeout(timeout)
          unsubscribe()
          reject(new Error(job.error || 'Job failed'))
        }
      })
    })
  }

  return { startJob }
}
```

**Step 2: Commit**

```bash
git add .
git commit -m "feat: add job queue hook"
```

---

### Task 2.3: Build the Draft/Edit/Learn component

**Files:**
- Create: `src/components/DraftEditor.tsx`
- Create: `src/hooks/useVoiceLearning.ts`

This is the core reusable component used across triage, nudges, meeting prep, meeting notes, and task delegation.

**Step 1: Create voice learning hook**

Create `src/hooks/useVoiceLearning.ts`:
```typescript
interface VoicePair {
  original_draft: string
  edited_version: string
  context: {
    recipient?: string
    channel?: string
    topic?: string
    tags: string[]
    source: 'triage' | 'nudge' | 'meeting_prep' | 'meeting_notes' | 'task'
  }
  created_at: string
}

export function useVoiceLearning() {
  const { user } = useAuth()

  const savePair = async (original: string, edited: string, context: VoicePair['context']) => {
    // Only save if meaningful edit (>5 chars changed)
    if (Math.abs(original.length - edited.length) < 5 && levenshteinDistance(original, edited) < 5) {
      return
    }
    const pairRef = push(ref(db, `users/${user.uid}/voice_pairs`))
    await set(pairRef, {
      original_draft: original,
      edited_version: edited,
      context,
      created_at: new Date().toISOString(),
    })
  }

  return { savePair }
}
```

**Step 2: Build DraftEditor component**

Create `src/components/DraftEditor.tsx`:

Props:
- `originalDraft: string` — the AI-generated draft
- `context: VoicePair['context']` — metadata for voice learning
- `onSave: (editedText: string) => void` — callback after save
- `onPushToGmail?: (editedText: string) => void` — optional Gmail draft button
- `loading?: boolean` — show skeleton while generating

Behavior:
- Renders a `<Textarea>` (shadcn) pre-filled with `originalDraft`
- Tracks user edits in local state
- **Save** button: calls `savePair()` if meaningful edits, then calls `onSave`
- **Push to Gmail Draft** button (conditional): calls `onPushToGmail`
- Visual indicator when draft has been edited (e.g., subtle border color change)

**Step 3: Test with mock data**

Render DraftEditor in the Meetings page with a hardcoded draft. Edit, save. Check Firebase console for voice_pairs entry.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: build Draft/Edit/Learn component with voice learning"
```

---

### Task 2.4: Build Meetings page — notes list view

**Files:**
- Create: `src/hooks/useMeetings.ts`
- Create: `src/components/meetings/MeetingNotesList.tsx`
- Create: `src/components/meetings/MeetingNoteCard.tsx`
- Modify: `src/pages/Meetings.tsx`

**Step 1: Create meetings hook**

Create `src/hooks/useMeetings.ts` — listens to `users/{uid}/meetings`, `users/{uid}/notes`, and `users/{uid}/transcripts`.

**Step 2: Build MeetingNotesList**

List of meeting notes with:
- Title, date, attendees
- Search filter (debounced text input)
- Date filter
- Status badges: "Pending" (no notes), "Notes" (has notes), "Notion" (uploaded)

**Step 3: Build MeetingNoteCard**

Card showing a single note:
- Title (editable inline)
- Date
- Content rendered as markdown (use a lightweight markdown renderer like `react-markdown`)
- Tabs: Notes | Transcript
- Actions: Upload to Notion, Sync from Notion

**Step 4: Wire up Meetings page**

Tab layout:
- "Notes" tab — MeetingNotesList showing processed meetings
- "Pending" tab — unprocessed meetings with "Generate Notes" button

**Step 5: Verify**

If you've migrated meeting data from the old project, verify notes display. Otherwise, test with manual Firebase entries.

**Step 6: Commit**

```bash
git add .
git commit -m "feat: build Meetings page with notes list"
```

---

### Task 2.5: Add transcript processing flow

**Files:**
- Create: `src/components/meetings/PasteTranscriptModal.tsx`
- Modify: `src/pages/Meetings.tsx`

**Step 1: Build PasteTranscriptModal**

Modal with:
- Large textarea for pasting raw transcript
- Meeting title input
- Meeting type selector: "Sales" | "Interview" (shadcn radio group or select)
- Submit button

On submit:
1. Write meeting to `users/{uid}/meetings/{id}` with transcript
2. Start job: `{ type: 'generate', meeting_id: id, meeting_type: type }`
3. Show loading state while job processes
4. On complete, navigate to the generated note

**Step 2: Add "Paste Transcript" button to Meetings page**

**Step 3: Test full flow**

Paste a test transcript. Select "Sales". Submit. Wait for Cloud Function to process. Verify notes appear.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add transcript paste and note generation flow"
```

---

### Task 2.6: Add Notion upload and sync

**Files:**
- Create: `src/components/meetings/NotionUploadButton.tsx`
- Modify: `src/components/meetings/MeetingNoteCard.tsx`

**Step 1: Build NotionUploadButton**

Dropdown button with "Team Hub" and "Personal Hub" options. On click:
1. Start job: `{ type: 'upload_notion', note_id: noteId, hub: 'team' | 'personal' }`
2. Show loading state
3. On complete, show "Uploaded" badge and "Open in Notion" link

**Step 2: Add Sync from Notion button**

For notes that have been uploaded (`notion_uploaded: true`), show a "Sync" button that:
1. Start job: `{ type: 'sync_notion', note_id: noteId }`
2. Refreshes note content on completion

**Step 3: Integrate DraftEditor for note editing**

Wrap meeting note content in the DraftEditor component so edits trigger voice learning (source: 'meeting_notes').

**Step 4: Test**

Upload a note to Notion. Verify it appears in the correct Notion database. Edit in Notion. Click Sync. Verify changes come back.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add Notion upload and sync for meeting notes"
```

---

### Task 2.7: Update database rules for meetings and voice pairs

**Files:**
- Modify: `database.rules.json`

**Step 1: Add validation rules**

Port meeting/notes/transcript/job rules from existing meeting notes app (`/Users/patrickstobbs/Documents/coding/personal/meeting_notes/database.rules.json`). Add `voice_pairs` rules.

**Step 2: Deploy and test**

```bash
firebase deploy --only database
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: add database rules for meetings, notes, voice pairs"
```

---

## Phase 3: Triage + Nudges

**Exit criteria:** Gmail OAuth working. Triage view: fetch inbox, classify by tier, generate draft replies, edit/save/push-to-Gmail-draft. Nudge view: detect stale threads, generate nudge drafts.

### Task 3.1: Set up Gmail OAuth

**Files:**
- Modify: `src/lib/firebase.ts`
- Create: `src/hooks/useGmail.ts`

**Step 1: Add Gmail scopes to Google provider**

Update `src/lib/firebase.ts`:
```typescript
googleProvider.addScope('https://www.googleapis.com/auth/gmail.readonly')
googleProvider.addScope('https://www.googleapis.com/auth/gmail.compose')
```

Note: Users will need to re-authenticate to grant the new scopes. Add a "Connect Gmail" flow if the token doesn't have Gmail scopes.

**Step 2: Create Gmail hook**

Create `src/hooks/useGmail.ts`:
- `fetchInbox(maxResults)` — GET recent emails via Gmail API
- `getThread(threadId)` — GET full thread
- `createDraft(to, subject, body, threadId?)` — POST Gmail draft

All calls go through `https://www.googleapis.com/gmail/v1/users/me/...` with the OAuth token.

**Step 3: Test**

Fetch inbox. Log results. Verify emails are returned.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add Gmail OAuth and API client"
```

---

### Task 3.2: Build triageInbox Cloud Function

**Files:**
- Create: `functions/triageInbox.js`
- Modify: `functions/index.js`

**Step 1: Build the function**

`triageInbox.js` — triggered by job queue (`type: 'triage'`):

1. Read Gmail OAuth token from the job params (passed from frontend)
2. Fetch recent emails via Gmail API (last 24h, unread, max 50)
3. For each email, extract: id, from, subject, snippet, date, threadId
4. Send all emails to Claude API with classification prompt:
   - Tier 1: Requires immediate response (board, key customers, urgent deadlines)
   - Tier 2: Handle today (team, active projects, time-sensitive)
   - Tier 3: FYI only (newsletters, notifications, low-priority)
5. For Tier 1 and 2 emails, generate draft replies using Claude API with voice pairs as few-shot examples
6. Write results to `users/{uid}/triage/{sessionId}/`
7. Mark job complete

**Note on OAuth token passing:** The frontend has the user's Google OAuth token. Pass it to the Cloud Function via the job params. The Cloud Function uses it to call Gmail API.

**Step 2: Query voice pairs helper**

Create `functions/voiceHelper.js` — shared utility for querying top 3-5 voice pairs from Firebase:
```javascript
async function getRelevantVoicePairs(db, uid, context) {
  const snap = await db.ref(`users/${uid}/voice_pairs`).once('value')
  const pairs = snap.val() || {}
  // Rank by: same recipient > same topic/tags > same source
  // Return top 5
}
```

**Step 3: Export and deploy**

```bash
firebase deploy --only functions
```

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add triageInbox Cloud Function"
```

---

### Task 3.3: Build Triage page

**Files:**
- Create: `src/components/triage/TriageInbox.tsx`
- Create: `src/components/triage/EmailCard.tsx`
- Create: `src/components/triage/TriageControls.tsx`
- Modify: `src/pages/Triage.tsx`

**Step 1: Build TriageControls**

Top bar with:
- "Run Triage" button (triggers triage job)
- Mode selector: Quick (Tier 1 only) | Full (all tiers)
- Loading indicator while triage runs

**Step 2: Build EmailCard**

Single email card showing:
- From, subject, date
- Tier badge (color-coded: red=T1, yellow=T2, gray=T3)
- Email snippet (expandable to full thread)
- DraftEditor component for the AI-generated reply
- "Push to Gmail Draft" button

**Step 3: Build TriageInbox**

List of EmailCards grouped by tier:
- Tier 1 section (expanded by default)
- Tier 2 section (expanded in full mode, collapsed in quick mode)
- Tier 3 section (collapsed, just subjects)
- Progress indicator: "3 of 8 emails handled"

**Step 4: Wire up Triage page**

On "Run Triage":
1. Get current Google OAuth token
2. Start job: `{ type: 'triage', mode: 'full', google_token: token }`
3. Listen for job completion
4. Render results from `users/{uid}/triage/{sessionId}/`

**Step 5: Test full flow**

Run triage. Verify emails appear classified by tier. Edit a draft. Save (check voice_pairs). Push to Gmail draft. Check Gmail for the draft.

**Step 6: Commit**

```bash
git add .
git commit -m "feat: build Triage page with inbox classification and draft editing"
```

---

### Task 3.4: Build pushToGmailDraft Cloud Function

**Files:**
- Create: `functions/pushToGmailDraft.js`
- Modify: `functions/index.js`

**Step 1: Build callable function**

`pushToGmailDraft.js` — HTTP callable function:
- Receives: `{ to, subject, body, threadId, google_token }`
- Creates Gmail draft via Gmail API using the user's OAuth token
- Returns: `{ draftId, draftUrl }`

Gmail API endpoint: `POST https://www.googleapis.com/gmail/v1/users/me/drafts`

Body format (RFC 2822):
```javascript
const raw = Buffer.from(
  `To: ${to}\r\nSubject: ${subject}\r\nContent-Type: text/plain; charset=utf-8\r\n\r\n${body}`
).toString('base64url')

// POST body: { message: { raw, threadId } }
```

**Step 2: Deploy and test**

```bash
firebase deploy --only functions
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: add pushToGmailDraft Cloud Function"
```

---

### Task 3.5: Build generateNudges Cloud Function

**Files:**
- Create: `functions/generateNudges.js`
- Modify: `functions/index.js`

**Step 1: Build the function**

`generateNudges.js` — triggered by job queue (`type: 'nudges'`):

1. Read Gmail OAuth token from job params
2. Fetch sent emails from last 14 days via Gmail API
3. For each sent email, check if a reply was received
4. Filter to threads where:
   - You sent the last message
   - No reply received within configurable window (default 3 days)
   - Not already nudged 2+ times
   - Not within 5-day cooldown since last nudge
5. For each stale thread, query Notion CRM for contact context
6. Generate nudge drafts via Claude API with voice pairs
7. Write results to `users/{uid}/nudges/`
8. Mark job complete

Reference existing nudge logic in `/Users/patrickstobbs/Documents/coding/personal/chief-of-staff/commands/nudge.md` for the rules and guardrails.

**Step 2: Deploy**

```bash
firebase deploy --only functions
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: add generateNudges Cloud Function"
```

---

### Task 3.6: Build Nudges page

**Files:**
- Create: `src/components/nudges/NudgeList.tsx`
- Create: `src/components/nudges/NudgeCard.tsx`
- Modify: `src/pages/Nudges.tsx`

**Step 1: Build NudgeCard**

Single nudge card showing:
- Thread subject
- Contact name + tier badge (from Notion CRM)
- Last activity date + "X days ago"
- Nudge count (0/2, 1/2)
- DraftEditor for the nudge draft
- "Push to Gmail Draft" button

**Step 2: Build NudgeList**

List of NudgeCards sorted by staleness (oldest first). Header with:
- "Scan for Nudges" button (triggers nudge job)
- Count: "5 follow-ups due"

**Step 3: Wire up Nudges page**

On "Scan for Nudges":
1. Get Google OAuth token
2. Start job: `{ type: 'nudges', google_token: token }`
3. Render results from `users/{uid}/nudges/`

**Step 4: Test full flow**

Run nudge scan. Verify stale threads appear. Edit a nudge. Push to Gmail draft.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: build Nudges page with stale thread detection"
```

---

### Task 3.7: Add Notion CRM query helper

**Files:**
- Create: `functions/notionCRM.js`

**Step 1: Build helper**

`notionCRM.js` — shared utility for querying contact data from Notion CRM:
- `getContactByEmail(apiKey, email)` — search Notion CRM database by email
- `getContactByName(apiKey, name)` — search by name
- Returns: `{ name, email, company, role, tier, notes, last_interaction }`

Used by triageInbox, generateNudges, and generateMeetingPrep functions.

**Step 2: Commit**

```bash
git add .
git commit -m "feat: add Notion CRM query helper for Cloud Functions"
```

---

## Phase 4: Meeting Prep

**Exit criteria:** Click a meeting on the Meetings page, generate prep notes that pull context from GCal, Gmail, Notion CRM, and past meeting notes.

### Task 4.1: Build generateMeetingPrep Cloud Function

**Files:**
- Create: `functions/generateMeetingPrep.js`
- Modify: `functions/index.js`

**Step 1: Build the function**

`generateMeetingPrep.js` — triggered by job queue (`type: 'meeting_prep'`):

1. Read meeting details from job params (title, attendees, time, google_token)
2. For each attendee, gather context:
   a. **Gmail:** Recent threads with this person (Gmail API search: `from:{email} OR to:{email}`, last 30 days)
   b. **GCal:** Past meetings with this person (GCal API, last 90 days)
   c. **Notion CRM:** Contact record (via notionCRM.js)
   d. **Meeting Notes:** Past notes mentioning this person (search Firebase notes)
3. Bundle all context into a Claude API prompt:
   - System: "You are preparing meeting prep notes for [user]. Be concise and actionable."
   - User: Meeting details + all gathered context
4. Write prep notes to `users/{uid}/meeting_prep/{meetingId}/`
5. Mark job complete

**Step 2: Deploy**

```bash
firebase deploy --only functions
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: add generateMeetingPrep Cloud Function"
```

---

### Task 4.2: Add meeting prep UI to Meetings page

**Files:**
- Create: `src/components/meetings/TodaysMeetings.tsx`
- Create: `src/components/meetings/MeetingPrepCard.tsx`
- Modify: `src/pages/Meetings.tsx`

**Step 1: Build TodaysMeetings**

Section at top of Meetings page showing today's calendar events:
- Fetched from GCal via useGoogleCalendar hook
- Each event shows: time, title, attendees
- "Prep" button on each meeting

**Step 2: Build MeetingPrepCard**

Expandable card showing prep notes for a meeting:
- Generated prep notes rendered as markdown
- Wrapped in DraftEditor for edit/learn (source: 'meeting_prep')
- Sources used badge (Gmail, GCal, Notion, Notes)

**Step 3: Update Meetings page layout**

Three sections:
1. **Today's Meetings** — TodaysMeetings component (top)
2. **Notes** — MeetingNotesList (existing)
3. **Pending** — unprocessed meetings

**Step 4: Test**

Generate prep for a real meeting. Verify context from multiple sources appears. Edit prep notes. Save (check voice_pairs).

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add meeting prep generation and UI"
```

---

## Phase 5: Dashboard + Delegate

**Exit criteria:** Dashboard loads instantly with pre-computed data. "Delegate to Claude" on tasks produces useful output.

### Task 5.1: Build refreshDashboard Cloud Function

**Files:**
- Create: `functions/refreshDashboard.js`
- Modify: `functions/index.js`

**Step 1: Build the function**

`refreshDashboard.js` — two triggers:
1. **Scheduled:** Runs at 7am Europe/London daily
2. **HTTP callable:** Manual refresh from frontend

Logic:
1. Read Google OAuth token (for scheduled: stored encrypted in user settings; for manual: passed in params)
2. Fetch today's meetings from GCal API
3. Read tasks from Firebase — filter to due today + overdue
4. Read nudges from Firebase — filter to due
5. Query Notion CRM for stale contacts (Tier 1 >14d, Tier 2 >30d, Tier 3 >60d)
6. Write to `users/{uid}/dashboard_cache/`:
   ```javascript
   {
     last_refreshed: new Date().toISOString(),
     meetings_today: [...],
     priority_tasks: [...],
     due_nudges: [...],
     stale_contacts: [...]
   }
   ```

**Step 2: Deploy**

```bash
firebase deploy --only functions
```

**Step 3: Commit**

```bash
git add .
git commit -m "feat: add refreshDashboard Cloud Function"
```

---

### Task 5.2: Build Dashboard page

**Files:**
- Create: `src/hooks/useDashboard.ts`
- Create: `src/components/dashboard/MeetingsWidget.tsx`
- Create: `src/components/dashboard/TasksWidget.tsx`
- Create: `src/components/dashboard/NudgesWidget.tsx`
- Create: `src/components/dashboard/ContactsWidget.tsx`
- Modify: `src/pages/Dashboard.tsx`

**Step 1: Create dashboard hook**

`useDashboard.ts` — listens to `users/{uid}/dashboard_cache/`. Returns cached data + `refresh()` function.

**Step 2: Build widgets**

Four compact card widgets:

**MeetingsWidget:** List of today's meetings with time, title, attendee count. Click navigates to `/meetings`.

**TasksWidget:** Priority tasks (due today + overdue). Shows count + top 3. Click navigates to `/tasks`.

**NudgesWidget:** Due nudges count + top 3 contacts needing follow-up. Click navigates to `/nudges`.

**ContactsWidget:** Stale contacts by tier. Shows names and days since last contact.

**Step 3: Compose Dashboard page**

Grid layout (2x2 on desktop, stacked on mobile). Refresh button in header. "Last updated: X" timestamp.

**Step 4: Test**

Trigger dashboard refresh. Verify all widgets populate. Click through to sub-pages.

**Step 5: Commit**

```bash
git add .
git commit -m "feat: build Dashboard page with widgets"
```

---

### Task 5.3: Build delegateTask Cloud Function

**Files:**
- Create: `functions/delegateTask.js`
- Modify: `functions/index.js`

**Step 1: Build the function**

`delegateTask.js` — triggered by job queue (`type: 'delegate'`):

1. Read task details from job params (task text, notes, google_token)
2. Call Claude API with tool use enabled:
   - Tools available:
     - `search_gmail(query)` — search Gmail via API
     - `read_email(messageId)` — read full email
     - `create_gmail_draft(to, subject, body)` — create draft
     - `search_calendar(query, startDate, endDate)` — search GCal
     - `query_notion(databaseId, filter)` — query Notion
   - System prompt: "You are an executive assistant. Complete this task. Return your output for user review."
   - User prompt: task description + notes
3. Run the agent loop (Claude decides which tools to call)
4. Write output to `users/{uid}/jobs/{jobId}/result`:
   - `{ output: string, actions_taken: string[], drafts_created: string[] }`
5. Mark job complete

**Step 2: Implement tool handlers**

Each tool calls the relevant API using the OAuth token (Gmail, GCal) or API key (Notion).

**Step 3: Deploy**

```bash
firebase deploy --only functions
```

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add delegateTask Cloud Function with tool use"
```

---

### Task 5.4: Add "Delegate to Claude" UI on Tasks page

**Files:**
- Create: `src/components/tasks/DelegateModal.tsx`
- Modify: `src/components/tasks/TaskItem.tsx`

**Step 1: Build DelegateModal**

Modal showing:
- Task text and notes (read-only, for context)
- "Delegate" button
- Loading state while agent runs (show "Claude is working..." with elapsed time)
- On complete: show output in DraftEditor (editable, with voice learning)
- Actions taken summary (e.g., "Searched Gmail for X", "Created draft to Y")

**Step 2: Add delegate button to TaskItem**

Small icon button on each task. Opens DelegateModal.

**Step 3: Test**

Create a task like "Draft email to John about Q2 planning". Click Delegate. Verify Claude runs the agent loop and returns a draft. Edit, save.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add Delegate to Claude for tasks"
```

---

## Phase 6: Polish + Retire

**Exit criteria:** Auto-task creation from meeting notes. Confidence-tested. Old apps retired when ready.

### Task 6.1: Build autoCreateTasks Cloud Function

**Files:**
- Create: `functions/autoCreateTasks.js`
- Modify: `functions/index.js`

**Step 1: Build the function**

`autoCreateTasks.js` — triggered by job queue (`type: 'auto_tasks'`):

1. Read meeting note content from job params
2. Call Claude API: "Extract action items from these meeting notes. Return as JSON array of { text, category ('business' | 'personal'), due_date? }"
3. Write suggested tasks to `users/{uid}/jobs/{jobId}/result`
4. Frontend shows suggestions — user approves/edits/dismisses each one
5. Approved tasks get added to the task list

**Step 2: Wire into meeting notes flow**

After note generation completes, automatically trigger `auto_tasks` job. Show "Suggested tasks" section below the note.

**Step 3: Deploy and test**

```bash
firebase deploy --only functions
```

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add auto-task creation from meeting notes"
```

---

### Task 6.2: Add follow-up email drafts from meeting notes

**Files:**
- Create: `functions/generateFollowUps.js`
- Create: `src/components/meetings/FollowUpDrafts.tsx`

**Step 1: Build Cloud Function**

`generateFollowUps.js` — triggered by job queue (`type: 'follow_ups'`):
- Read meeting notes
- Call Claude API: identify attendees who need follow-up emails
- Generate draft follow-ups with voice pairs for style
- Return drafts for review

**Step 2: Build UI**

Show follow-up drafts below meeting notes. Each draft in a DraftEditor. "Push to Gmail Draft" button per draft.

**Step 3: Commit**

```bash
git add .
git commit -m "feat: add follow-up email drafts from meeting notes"
```

---

### Task 6.3: PWA setup

**Files:**
- Create: `public/manifest.json`
- Create: `public/sw.js`
- Modify: `index.html`

**Step 1: Add web app manifest**

Standard PWA manifest with app name, icons, theme color.

**Step 2: Add service worker**

Cache app shell for offline loading. Network-first for Firebase API calls.

**Step 3: Test**

Verify "Install" prompt appears in Chrome. Install. Verify app works as standalone window.

**Step 4: Commit**

```bash
git add .
git commit -m "feat: add PWA support"
```

---

### Task 6.4: Data migration from old projects

**Only do this when confident the new app is working.**

**Step 1: Export todo data**

```bash
firebase --project paddy-s-todo-list database:get /users > /tmp/todo-export.json
```

**Step 2: Export meeting notes data**

```bash
firebase --project meeting-notes-paddy database:get /users > /tmp/notes-export.json
```

**Step 3: Merge and import**

Write a script that merges both exports (tasks from todo, meetings/notes from meeting notes) and imports into the new project:

```bash
firebase --project chief-of-staff-paddy database:set /users --data /tmp/merged.json
```

**Step 4: Verify**

Check all tasks, meetings, notes appear correctly in the new app.

---

### Task 6.5: Retire old apps

**Only do this after confident migration and testing.**

1. Update DNS/bookmarks to point to new app
2. Add redirect from old Firebase Hosting URLs to new app
3. Archive old Git repos (don't delete)
4. Keep old Firebase projects alive for 30 days as safety net
5. After 30 days with no issues, disable old projects

---

## Appendix: File Structure (Final)

```
chief-of-staff-app/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── firebase.json
├── .firebaserc
├── database.rules.json
├── public/
│   ├── manifest.json
│   └── sw.js
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── lib/
│   │   ├── firebase.ts
│   │   └── utils.ts
│   ├── types/
│   │   └── task.ts
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useFirebase.ts
│   │   ├── useTasks.ts
│   │   ├── useJobs.ts
│   │   ├── useVoiceLearning.ts
│   │   ├── useGoogleCalendar.ts
│   │   ├── useGmail.ts
│   │   ├── useMeetings.ts
│   │   ├── useDashboard.ts
│   │   └── useDragDrop.ts
│   ├── components/
│   │   ├── ui/                    (shadcn components)
│   │   ├── layout/
│   │   │   ├── AppShell.tsx
│   │   │   └── Sidebar.tsx
│   │   ├── DraftEditor.tsx        (core Draft/Edit/Learn component)
│   │   ├── LoginPage.tsx
│   │   ├── tasks/
│   │   │   ├── TaskList.tsx
│   │   │   ├── TaskItem.tsx
│   │   │   ├── TaskInput.tsx
│   │   │   ├── TaskNotes.tsx
│   │   │   ├── TaskScheduler.tsx
│   │   │   ├── CalendarView.tsx
│   │   │   ├── DividerRow.tsx
│   │   │   └── DelegateModal.tsx
│   │   ├── triage/
│   │   │   ├── TriageInbox.tsx
│   │   │   ├── EmailCard.tsx
│   │   │   └── TriageControls.tsx
│   │   ├── nudges/
│   │   │   ├── NudgeList.tsx
│   │   │   └── NudgeCard.tsx
│   │   ├── meetings/
│   │   │   ├── MeetingNotesList.tsx
│   │   │   ├── MeetingNoteCard.tsx
│   │   │   ├── TodaysMeetings.tsx
│   │   │   ├── MeetingPrepCard.tsx
│   │   │   ├── PasteTranscriptModal.tsx
│   │   │   ├── NotionUploadButton.tsx
│   │   │   └── FollowUpDrafts.tsx
│   │   └── dashboard/
│   │       ├── MeetingsWidget.tsx
│   │       ├── TasksWidget.tsx
│   │       ├── NudgesWidget.tsx
│   │       └── ContactsWidget.tsx
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── Triage.tsx
│   │   ├── Tasks.tsx
│   │   ├── Nudges.tsx
│   │   └── Meetings.tsx
│   └── styles/
│       └── globals.css
└── functions/
    ├── package.json
    ├── index.js
    ├── generateNotes.js           (ported from meeting notes app)
    ├── uploadToNotion.js          (ported from meeting notes app)
    ├── syncFromNotion.js          (ported from meeting notes app)
    ├── notionHelpers.js           (ported from meeting notes app)
    ├── voiceHelper.js             (shared voice pair querying)
    ├── notionCRM.js               (shared Notion CRM querying)
    ├── triageInbox.js
    ├── generateDraft.js
    ├── generateNudges.js
    ├── generateMeetingPrep.js
    ├── delegateTask.js
    ├── pushToGmailDraft.js
    ├── refreshDashboard.js
    ├── autoCreateTasks.js
    └── generateFollowUps.js
```
