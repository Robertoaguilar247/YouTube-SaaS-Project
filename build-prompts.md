# TaskFlow — Build Prompts

These prompts are designed to be run sequentially, each building on the output of the last. Each prompt is self-contained with full context so the AI never has to guess what came before.

---

## Prompt 1 — Project Scaffold & Database Schema

```
Create the initial scaffold for a Node.js + Express SaaS todo list app called TaskFlow.

Set up the following:

1. A package.json with these dependencies only:
   - express
   - express-session
   - better-sqlite3
   - bcrypt
   - connect-sqlite3 (session store)
   - dotenv

2. A server/index.js that:
   - Loads .env
   - Initialises express-session with a SQLite store, 7-day maxAge, HttpOnly + SameSite=Strict cookies
   - Mounts routes from server/routes/auth.js and server/routes/tasks.js
   - Serves static files from the project root
   - Listens on PORT from .env (default 3000)

3. A server/db/schema.sql with two tables:

   users:
   - id INTEGER PRIMARY KEY AUTOINCREMENT
   - email TEXT UNIQUE NOT NULL
   - password_hash TEXT NOT NULL
   - display_name TEXT
   - failed_attempts INTEGER DEFAULT 0
   - locked_until INTEGER
   - created_at INTEGER DEFAULT (unixepoch())

   tasks:
   - id INTEGER PRIMARY KEY AUTOINCREMENT
   - user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE
   - title TEXT NOT NULL
   - description TEXT
   - status TEXT NOT NULL DEFAULT 'todo' CHECK(status IN ('todo','in_progress','review','done'))
   - priority TEXT NOT NULL DEFAULT 'none' CHECK(priority IN ('none','low','medium','high'))
   - due_date TEXT
   - created_at INTEGER DEFAULT (unixepoch())
   - updated_at INTEGER DEFAULT (unixepoch())

4. A server/db/db.js that opens the SQLite database, runs schema.sql on first boot, and exports the db instance.

5. A .env.example with: SESSION_SECRET, PORT, NODE_ENV

Keep everything as simple as possible. No TypeScript, no ORM, no build step.
```

---

## Prompt 2 — Authentication Routes

```
We have a Node.js + Express app with SQLite (via better-sqlite3). The db instance is exported from server/db/db.js. Sessions are configured in server/index.js using express-session.

Create server/routes/auth.js with the following routes. All responses are JSON.

POST /auth/register
- Accept { email, display_name, password }
- Validate: email is valid format, password is at least 8 characters
- Check email is not already taken (return 409 if so)
- Hash password with bcrypt (cost 12)
- Insert user into users table
- Set req.session.userId to the new user's id
- Return 201 { id, email, display_name }

POST /auth/login
- Accept { email, password }
- Look up user by email; return 401 if not found
- Check locked_until — if current time < locked_until, return 423 with seconds remaining
- Compare password with bcrypt
- On failure: increment failed_attempts; if >= 10, set locked_until to now + 15 minutes; return 401
- On success: reset failed_attempts to 0, set req.session.userId, return 200 { id, email, display_name }

POST /auth/logout
- Destroy session, return 200 { ok: true }

POST /auth/forgot-password
- Accept { email }
- Always return 200 { ok: true } (do not leak whether email exists)
- If email exists: generate a 32-byte random hex token, store it in a password_reset_tokens table
  (create this table: id, user_id, token TEXT UNIQUE, expires_at INTEGER, used INTEGER DEFAULT 0)
- Log the reset URL to the console for now: http://localhost:3000/reset-password.html?token=TOKEN

POST /auth/reset-password
- Accept { token, password }
- Look up token in password_reset_tokens where used = 0 and expires_at > now
- Return 400 if not found or expired
- Hash new password, update user, mark token used = 1
- Set req.session.userId, return 200 { ok: true }

DELETE /auth/account
- Require session (return 401 if not logged in)
- Delete the user from users table (CASCADE handles tasks)
- Destroy session
- Return 200 { ok: true }

Add a simple middleware function requireAuth(req, res, next) and export it — it returns 401 if req.session.userId is not set. Tasks routes will import and use this.
```

---

## Prompt 3 — Task API Routes

```
We have a Node.js + Express app with SQLite (via better-sqlite3). The db instance is exported from server/db/db.js. requireAuth middleware is exported from server/routes/auth.js.

Create server/routes/tasks.js with full CRUD for tasks. All routes require authentication via requireAuth. All responses are JSON. Never return tasks belonging to a different user than the session user.

GET /api/tasks
- Optional query params: status, priority (filter), sort (due_date | priority | created_at, default created_at), order (asc | desc, default desc)
- Return array of all tasks for the session user matching filters

POST /api/tasks
- Accept { title, description, status, priority, due_date }
- title is required (return 400 if missing or empty)
- status defaults to 'todo', priority defaults to 'none'
- Insert and return the created task as 201

PATCH /api/tasks/:id
- Accept any subset of { title, description, status, priority, due_date }
- Verify task belongs to session user (return 404 if not found or not owned)
- Update only the provided fields plus updated_at = unixepoch()
- Return the updated task

DELETE /api/tasks/:id
- Verify task belongs to session user (return 404 if not found or not owned)
- Hard delete, return 200 { ok: true }

GET /api/tasks/stats
- Return counts for the session user: { total, completed, in_progress, overdue }
- overdue = tasks where due_date < today's date AND status != 'done'

Keep all queries as simple prepared statements. No query builder, no ORM.
```

---

## Prompt 4 — Auth Pages (Login & Register)

```
Create two standalone HTML pages for a SaaS app called TaskFlow. Use Bootstrap 5.3 from CDN. Use IBM Plex Sans from Google Fonts. No other dependencies.

Design direction: clean centered card layout, warm off-white background (#f5f4f0), strong blue primary (#2563eb). The card should have no border and a subtle box shadow: 0 1px 3px rgba(0,0,0,.08), 0 4px 12px rgba(0,0,0,.06).

--- login.html ---
A centered login card (max-width 400px) with:
- "TaskFlow" as a small heading above the card
- Email input
- Password input
- "Log in" primary button (full width)
- Link below: "Don't have an account? Register" → register.html
- Link below: "Forgot your password?" → reset-password.html

On submit: POST /auth/login with { email, password } as JSON.
- On 200: redirect to index.html
- On 401: show inline error "Invalid email or password"
- On 423: show inline error "Account locked. Try again in X minutes."
- Disable the button and show a spinner while the request is in flight.

--- register.html ---
Same card layout with:
- Display name input
- Email input
- Password input (with helper text: "Minimum 8 characters")
- "Create account" primary button (full width)
- Link below: "Already have an account? Log in" → login.html

On submit: POST /auth/register with { display_name, email, password } as JSON.
- On 201: redirect to index.html
- On 409: show inline error "An account with this email already exists"
- On 400: show inline error from server response
- Disable button + spinner while in flight.

No frameworks. Vanilla JS fetch for the API calls. Keep JS inline in a <script> tag at the bottom of each file.
```

---

## Prompt 5 — App Shell (Sidebar + Topbar + Shared CSS)

```
Create two shared files used by all authenticated pages in TaskFlow.

--- assets/css/app.css ---
CSS custom properties layered on Bootstrap 5.3 (do not import Bootstrap here — it's loaded via CDN in each HTML file):

:root {
  --bs-body-bg: #f5f4f0;
  --bs-primary: #2563eb;
  --bs-primary-rgb: 37, 99, 235;
  --app-sidebar-bg: #1e1e2e;
  --app-sidebar-color: #cdd6f4;
  --app-card-shadow: 0 1px 3px rgba(0,0,0,.08), 0 4px 12px rgba(0,0,0,.06);
}

body { font-family: 'IBM Plex Sans', sans-serif; }
h1, h2, h3, h4, h5, h6 { font-family: 'DM Sans', sans-serif; }
.font-mono { font-family: 'JetBrains Mono', monospace; }

Also add:
- Sidebar nav link styles: active state uses --bs-primary background at 15% opacity with primary text color; hover state is white at 8% opacity
- A utility class .card-app that applies border-0 and --app-card-shadow
- Ensure focus outlines are never suppressed (do not set outline: none anywhere)

--- assets/js/app.js ---
Shared JS loaded on every authenticated page:

1. Auth guard: on DOMContentLoaded, call GET /api/tasks/stats. If response is 401, redirect to login.html immediately. (This is a lightweight way to check session validity without a dedicated /auth/me endpoint.)

2. Logout: find any element with data-action="logout", add click handler that POSTs to /auth/logout then redirects to login.html.

3. Toast helper: expose window.showToast(message, type='success') that programmatically creates and shows a Bootstrap Toast in a container fixed to bottom-end. type can be 'success', 'danger', or 'info'. Auto-dismisses after 3 seconds.

4. Initialize Bootstrap tooltips globally:
   document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(el => new bootstrap.Tooltip(el))

Then create a reusable HTML snippet (as a comment block at the top of app.js, clearly labelled "SHELL SNIPPET — paste into each authenticated page") containing:

- The sidebar: dark background, TaskFlow logo text, nav links for Dashboard / Kanban / Calendar / My Tasks using Bootstrap Icons (bi-grid-1x2, bi-kanban, bi-calendar3, bi-check2-square). Each link is a plain <a> tag. Active link gets class "active".
- The topbar: sticky-top, page title as an h1.h5, search input (max 260px), "+ New Task" button (btn-primary btn-sm), and a small avatar placeholder div (32x32 rounded-circle bg-secondary) with data-action="logout" on a dropdown or direct click.
- The outer layout wrapper: d-flex, sidebar fixed left, main content flex-grow-1 with the topbar at top and a <main> content area below.
```

---

## Prompt 6 — Dashboard Page

```
Create index.html for the TaskFlow dashboard. This is a protected page — it uses the shared shell from assets/css/app.css and assets/js/app.js (both loaded via <script> and <link> tags). Bootstrap 5.3 and Bootstrap Icons are loaded from CDN. Google Fonts: DM Sans + IBM Plex Sans.

Paste in the shell snippet from app.js (sidebar + topbar + layout wrapper). Set the active nav link to Dashboard. Set the topbar title to "Dashboard".

Inside <main class="p-4">:

1. Stat cards row (col-sm-6 col-xl-3, g-3):
   Four cards: Total Tasks, Completed, In Progress, Overdue.
   Each card has a small label, a large bold number, and a subtle badge.
   Use class card-app on each card.

2. Progress bar (below stat cards, mb-4):
   A label "Task Distribution" and a Bootstrap stacked progress bar (height 8px) with four segments: done (success), in_progress (primary), todo (warning), overdue (danger).

3. Today's tasks card (card-app):
   Header with "Today" label and "View all" link to tasks.html.
   A Bootstrap list-group-flush inside. Each item has a checkbox, task title, and priority badge.
   If no tasks due today, show a muted empty state: "Nothing due today."

On DOMContentLoaded (after the auth guard in app.js runs):
- GET /api/tasks/stats → populate the four stat card numbers and the progress bar widths
- GET /api/tasks?due_date=TODAY&sort=priority → populate today's list (TODAY = current date as YYYY-MM-DD)
- Checking a checkbox should PATCH /api/tasks/:id with { status: 'done' } and call showToast('Task completed')
- "+ New Task" button opens the task creation modal (defined below)

Include the task creation modal (Bootstrap Modal, id="newTaskModal"):
- Title input (required)
- Description textarea
- Priority select (None / Low / Medium / High)
- Due date input
- Cancel + Create Task buttons
On Create Task: POST /api/tasks, on 201 re-fetch stats and today's list, show showToast('Task created'), close modal.
```

---

## Prompt 7 — Kanban Board Page

```
Create kanban.html for the TaskFlow Kanban board. Load the shared shell (assets/css/app.css, assets/js/app.js), Bootstrap 5.3, Bootstrap Icons, and Sortable.js — all from CDN. Set active nav link to Kanban. Topbar title: "Kanban".

Inside <main>: a full-height horizontally scrollable container (d-flex gap-3 overflow-auto p-4 align-items-start).

Four columns — To Do, In Progress, Review, Done — each 300px wide (flex-shrink-0). Each column is a card-app with:
- Header: column name + task count badge (bg-secondary-subtle)
- Body (id="col-todo" / col-in-progress / col-review / col-done): where cards are dropped
- Footer: "+ Add task" button (btn-outline-secondary btn-sm w-100) that opens the task creation modal pre-set to that column's status

Each task card (draggable) shows:
- Priority badge (top-left) using the standard subtle color classes
- Due date (top-right, muted, with bi-calendar2 icon) — omit if null
- Task title (fw-medium)
- A subtle bottom row with task id in monospace (font-mono text-secondary small)

On DOMContentLoaded:
- GET /api/tasks → group tasks by status → render into the correct column
- Initialise Sortable on each column body with { group: 'tasks', animation: 150 }
- On Sortable's onEnd event: PATCH /api/tasks/:id with the new status derived from the drop target column id, call showToast('Task moved')
- Clicking a card (not dragging) opens the task detail offcanvas

Include the task detail offcanvas (offcanvas-end, 380px, id="taskDetail"):
- Editable title input
- Editable description textarea  
- Priority select
- Due date input
- Save button → PATCH /api/tasks/:id → showToast('Saved')
- Delete button → window.confirm → DELETE /api/tasks/:id → remove card from DOM → showToast('Task deleted', 'danger') → hide offcanvas

Include the task creation modal (same as dashboard, reuse the same HTML structure).
```

---

## Prompt 8 — My Tasks Page

```
Create tasks.html for the TaskFlow "My Tasks" list view. Load shared shell, Bootstrap 5.3, Bootstrap Icons from CDN. Set active nav link to My Tasks. Topbar title: "My Tasks".

Inside <main class="p-4">:

A filter/sort toolbar (d-flex gap-2 mb-3 flex-wrap):
- Status filter: All / To Do / In Progress / Review / Done (btn-group, button per option)
- Priority filter: All / High / Medium / Low (btn-group)
- Sort select: Due Date / Priority / Created (form-select form-select-sm w-auto)

A task table (table table-hover align-middle):
Columns: [ checkbox | Title | Priority | Status | Due Date | Actions ]
- Checkbox: PATCH status to 'done' on check
- Priority: badge with standard subtle colors
- Status: plain text, muted
- Due Date: formatted as "Feb 22" — red text if overdue and not done
- Actions: a small edit icon (bi-pencil) that opens the task detail offcanvas

Empty state: if no tasks match filters, show a centered muted message "No tasks found."

On DOMContentLoaded:
- GET /api/tasks with current filter/sort params → render rows
- Filter/sort controls re-fetch on change (debounced 200ms for the sort select)
- "+ New Task" in topbar opens the task creation modal; on success re-fetch the list

Reuse the same task detail offcanvas and task creation modal HTML structure from kanban.html — they are identical. Keep JS inline in a <script> at the bottom.
```

---

## Prompt 9 — Calendar Page

```
Create calendar.html for the TaskFlow calendar view. Load shared shell, Bootstrap 5.3, Bootstrap Icons from CDN. Set active nav link to Calendar. Topbar title: "Calendar".

Inside <main class="p-4">:

A card-app containing:
- Header: left chevron button, "February 2026" month/year label (fw-semibold), right chevron button
- Body: a 7-column CSS grid calendar

Day header row: Sun Mon Tue Wed Thu Fri Sat (text-secondary small fw-medium, text-center)

Day cells (min-height 90px, border, rounded-1, p-1):
- Day number in the top-left (small text-secondary)
- Days outside the current month: muted background (bg-light), greyed day number
- Today's date: day number has a small primary-colored circle background
- Task chips: for each task due on that day, a small full-width rounded chip showing truncated title. Color by priority: high=danger-subtle, medium=warning-subtle, low=success-subtle, none=secondary-subtle. Max 3 chips per day; if more, show "+N more" muted text
- Clicking a chip opens the task detail offcanvas

On DOMContentLoaded:
- GET /api/tasks → filter to tasks with a due_date → group by due_date → render chips
- Chevron buttons re-render the grid for the adjacent month (re-use the same task data, no extra fetch needed unless month changes significantly — just re-fetch on month navigation)

On mobile (< md breakpoint via JS window.innerWidth check): hide the grid and show a simple date-grouped list instead: date as a small header, tasks as list items with priority badge.

Include the same task detail offcanvas and task creation modal as other pages.
```

---

## Prompt 10 — Password Reset Page & Final Wiring

```
Complete the TaskFlow app with the final page and production hardening.

--- reset-password.html ---
Same card layout as login.html (centered, 400px max-width, warm background, no-border card with shadow).

Step 1 — Forgot password form (shown when no ?token= param in URL):
- Email input
- "Send reset link" button
- On submit: POST /auth/forgot-password { email }
- On 200: replace form with a success message: "If that email is registered, you'll receive a reset link shortly. Check the server console during development."

Step 2 — Reset password form (shown when ?token= is present in URL):
- New password input (min 8 characters)
- Confirm password input
- "Set new password" button
- Client-side validate passwords match before submitting
- On submit: POST /auth/reset-password { token, password }
- On 200: redirect to index.html
- On 400: show "This reset link is invalid or has expired."

--- server/index.js additions ---
Add these middleware additions before routes:
1. Helmet-lite (manually set headers, no extra package): set X-Content-Type-Options: nosniff, X-Frame-Options: DENY, Referrer-Policy: same-origin
2. A simple rate limiter for auth routes using a plain in-memory Map (no package needed): max 20 requests per IP per minute on /auth/* routes; return 429 if exceeded
3. A middleware that redirects /index.html, /kanban.html, /calendar.html, /tasks.html to /login.html if the session userId is not set — so protected pages can't be viewed without auth even without JS

--- Final checklist ---
Confirm the following work end-to-end:
1. Register → land on dashboard
2. Log out → redirected to login
3. Log in → land on dashboard, stats load
4. Create task → appears in dashboard today list, kanban, and my tasks
5. Drag task in kanban → status updates
6. Delete task → gone from all views
7. Visit dashboard URL directly while logged out → redirected to login.html
```

---

*These 10 prompts build the complete v1 application. Run them in order, paste the output of each into your project before running the next.*
