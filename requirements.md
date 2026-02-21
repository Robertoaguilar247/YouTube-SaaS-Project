# TaskFlow — Product Requirements Document

**Version:** 1.1 (Simplified)
**Last Updated:** February 2026
**Status:** Draft

---

## 1. Overview

TaskFlow is a simple SaaS todo list application. Users register, log in, and manage their personal tasks across a Dashboard, Kanban Board, and Calendar view. The guiding principle at every decision point is: **choose the simplest option that works.**

---

## 2. Goals

- Users can create, update, complete, and delete tasks.
- Tasks are private to each user account.
- The app works well on desktop and mobile browsers.
- Authentication is secure but not over-engineered.

---

## 3. Tech Stack

| Concern | Choice | Rationale |
|---|---|---|
| Frontend | HTML5 + Bootstrap 5.3 + vanilla JS | No build step, no framework overhead |
| Icons | Bootstrap Icons (CDN) | Already bundled with Bootstrap |
| Drag-and-drop | Sortable.js (CDN) | Lightweight, no dependencies |
| Backend | Node.js + Express | Minimal boilerplate, large ecosystem |
| Auth | express-session + bcrypt | Simple cookie sessions, no JWT complexity |
| Database | SQLite (dev) → PostgreSQL (prod) | Zero config locally, easy to promote |
| Hosting | Render or Railway | Free tier, one-click deploys, no DevOps |
| Email | Resend (free tier) | Simple API, minimal setup |

No Redis. No CDN configuration. No Docker required to get started.

---

## 4. User Authentication

### 4.1 Registration

- Email + password only. No OAuth, no social login.
- Password minimum: 8 characters. No complexity rules beyond that.
- Email confirmation is **not** required in v1 — users land on the Dashboard immediately after registering.

### 4.2 Login

- Email + password form.
- Session cookie set on success (HttpOnly, Secure in production).
- After 10 failed attempts, account is locked for 15 minutes.
- No "Remember me" checkbox — session lasts 7 days by default.

### 4.3 Password Reset

- User enters email, receives a reset link valid for 1 hour.
- Reset link contains a signed token stored in the database.
- On use, the token is invalidated and the user is logged in automatically.

### 4.4 Sessions

- Server-side sessions via express-session with a database store.
- One active session per user (new login invalidates prior session).
- Logout clears the session immediately.

### 4.5 Account Settings

- User can update display name and password.
- No email change, no avatar upload, no session management UI in v1.
- Account deletion: user can delete their account; all tasks are hard-deleted immediately.

---

## 5. Application Views

### 5.1 Shell Layout

Fixed sidebar (240px, desktop) with offcanvas drawer on mobile. Sticky topbar with page title, search input, and "+ New Task" button. User avatar in the topbar links to account settings.

Navigation links: Dashboard, Kanban, Calendar, My Tasks.

### 5.2 Dashboard

Default view after login. Shows:

- Four stat cards: Total Tasks, Completed, In Progress, Overdue.
- A stacked progress bar showing task distribution by status.
- A "Today" card listing tasks due today with inline completion checkboxes.

### 5.3 Kanban Board

Four fixed columns: To Do, In Progress, Review, Done. Tasks are draggable between columns via Sortable.js. Each card shows title, priority badge, and due date. An "+ Add task" button at the bottom of each column opens the task creation modal.

### 5.4 Calendar

Monthly grid view. Tasks appear as colored chips on their due date. Clicking a chip opens the task detail panel. Navigation arrows move between months. On mobile, degrades to a simple date-grouped list.

### 5.5 My Tasks

Flat list of all the user's tasks. Sortable by due date or priority. Filterable by status. Inline completion checkbox on each row.

---

## 6. Task Management

### 6.1 Task Data Model

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Auto-increment |
| `user_id` | integer | Foreign key to users table |
| `title` | string | Required, max 255 characters |
| `description` | text | Optional |
| `status` | enum | `todo`, `in_progress`, `review`, `done` |
| `priority` | enum | `none`, `low`, `medium`, `high` |
| `due_date` | date | Optional |
| `created_at` | timestamp | Auto-set |
| `updated_at` | timestamp | Auto-updated |

### 6.2 Task Creation

The "+ New Task" button opens a modal with: title (required), description (optional), priority dropdown, and due date picker. One Create button, one Cancel button.

### 6.3 Task Editing

Clicking a task opens a right-side offcanvas detail panel. All fields are editable. A Save button commits changes.

### 6.4 Task Deletion

Delete button in the detail panel. A simple confirm dialog (`window.confirm`) is sufficient in v1 — no soft delete, no undo.

### 6.5 Task Completion

Inline checkbox in Dashboard and My Tasks views, or drag to Done in Kanban. A brief toast confirms the action.

---

## 7. UI Patterns

- **Toasts:** Bootstrap Toasts, bottom-right, auto-dismiss after 3 seconds. Used for task created, saved, deleted, and errors.
- **Modals:** Bootstrap Modal for task creation only.
- **Offcanvas:** Bootstrap Offcanvas (right side, 380px) for task detail/edit.
- **Priority badges:** Bootstrap subtle color classes (danger/warning/success/secondary).
- **Dark mode:** Not in v1. Add later with a single `data-bs-theme` toggle.

---

## 8. Design Tokens

A small set of CSS variables layered on Bootstrap 5.3:

```css
:root {
  --bs-body-bg: #f5f4f0;
  --bs-primary: #2563eb;
  --app-sidebar-bg: #1e1e2e;
  --app-sidebar-color: #cdd6f4;
  --app-card-shadow: 0 1px 3px rgba(0,0,0,.08), 0 4px 12px rgba(0,0,0,.06);
}
```

Fonts: DM Sans (headings), IBM Plex Sans (body), JetBrains Mono (IDs/timestamps) — all via Google Fonts CDN.

---

## 9. Responsive Behavior

| Breakpoint | Sidebar | Kanban | Calendar |
|---|---|---|---|
| < md | Offcanvas drawer | Single-column scroll | Date-grouped list |
| md | Icon-only | 2-column scroll | Monthly grid |
| lg+ | Full 240px | All columns | Full grid + detail panel |

Handled entirely with Bootstrap utility classes — no custom JS for layout switching.

---

## 10. Accessibility

- Visible focus styles on all interactive elements (do not suppress `outline`).
- `aria-label` on all icon-only buttons.
- Color is never the sole indicator of state — always paired with text or icon.
- Target WCAG 2.1 AA compliance.

---

## 11. Security

- HTTPS enforced in production.
- Passwords hashed with bcrypt (cost factor 12).
- Sessions use HttpOnly, Secure, SameSite=Strict cookies.
- All user input sanitized server-side before persistence.
- Task API routes verify that the requesting user owns the task before any read/write.

---

## 12. Performance

- Target LCP under 2 seconds on broadband.
- No heavy client-side framework — Bootstrap + vanilla JS keeps the bundle small.
- Static assets cached via standard HTTP headers.

---

## 13. Out of Scope for v1

- Teams, workspaces, or task sharing
- File attachments
- Email or push notifications
- Real-time collaboration
- Native mobile apps
- Reporting or data exports
- OAuth / social login
- Dark mode toggle
- Email verification on signup
- Custom Kanban columns

---

## 14. File Structure

```
/
├── index.html            # Dashboard
├── kanban.html
├── calendar.html
├── tasks.html
├── login.html            # Public
├── register.html         # Public
├── reset-password.html   # Public
├── assets/
│   ├── css/app.css
│   └── js/app.js
└── server/
    ├── index.js          # Express entry point
    ├── routes/
    │   ├── auth.js
    │   └── tasks.js
    └── db/
        └── schema.sql
```

---

*End of Requirements Document*
