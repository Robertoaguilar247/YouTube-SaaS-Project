# Design Notes — Todo List Application

## Overview

A productivity-focused web application built with **Bootstrap 5.3**. The app includes a Dashboard, Kanban Board, Calendar, and supporting UI elements. The aesthetic direction is **refined utilitarian** — clean, high-contrast, purposeful. Every element earns its place.

---

## Tech Stack & Dependencies

- **Bootstrap 5.3** — layout, components, utilities
- **Bootstrap Icons** — iconography (`https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.css`)
- **FullCalendar** or a lightweight custom calendar grid for the calendar view
- **Sortable.js** — drag-and-drop for Kanban
- Custom CSS variables layered on top of Bootstrap's `:root` tokens

---

## Color & Theme

We extend Bootstrap 5.3's CSS variable system rather than overriding Sass. This keeps theming flexible and dark-mode ready.

```css
:root {
  --bs-body-bg: #f5f4f0;           /* warm off-white, not pure white */
  --bs-body-color: #1a1a1a;
  --bs-primary: #2563eb;           /* strong blue — actions, CTAs */
  --bs-primary-rgb: 37, 99, 235;
  --bs-secondary: #64748b;         /* slate — labels, meta text */
  --bs-success: #16a34a;
  --bs-warning: #d97706;
  --bs-danger: #dc2626;
  --app-sidebar-bg: #1e1e2e;       /* dark sidebar against warm body */
  --app-sidebar-color: #cdd6f4;
  --app-card-shadow: 0 1px 3px rgba(0,0,0,.08), 0 4px 12px rgba(0,0,0,.06);
}
```

Dark mode toggles via `data-bs-theme="dark"` on `<html>` — Bootstrap 5.3 handles the rest.

---

## Typography

- **Display / Headings**: `DM Sans` (Google Fonts) — geometric, modern, readable at small sizes
- **Body**: Bootstrap default stack is acceptable, but `Inter` may be swapped for `IBM Plex Sans` to add subtle personality
- **Monospace** (task IDs, timestamps): `JetBrains Mono`

```html
<link href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=IBM+Plex+Sans:wght@400;500&family=JetBrains+Mono&display=swap" rel="stylesheet">
```

Set in CSS:
```css
body { font-family: 'IBM Plex Sans', var(--bs-font-sans-serif); }
h1, h2, h3, h4, h5, h6 { font-family: 'DM Sans', var(--bs-font-sans-serif); }
```

---

## Layout & Navigation

### Shell Structure

```
┌──────────────────────────────────────────────┐
│  Sidebar (fixed, 240px)  │  Main Content Area │
│                          │  ┌──────────────┐  │
│  Logo                    │  │  Topbar      │  │
│  ─────────               │  ├──────────────┤  │
│  Nav links               │  │  Page View   │  │
│  ─────────               │  │  (Dashboard  │  │
│  Workspaces              │  │  / Kanban    │  │
│  ─────────               │  │  / Calendar) │  │
│  User / Settings         │  └──────────────┘  │
└──────────────────────────────────────────────┘
```

### Sidebar

Based on Bootstrap's [offcanvas](https://getbootstrap.com/docs/5.3/components/offcanvas/) for mobile, static column on desktop.

```html
<!-- Desktop sidebar uses d-flex flex-column on a col-auto -->
<nav class="d-flex flex-column flex-shrink-0 p-3 vh-100" style="width: 240px; background: var(--app-sidebar-bg);">
  <a href="/" class="d-flex align-items-center mb-4 text-decoration-none">
    <span class="fs-5 fw-semibold text-white">TaskFlow</span>
  </a>
  <ul class="nav nav-pills flex-column mb-auto gap-1">
    <li class="nav-item">
      <a href="#" class="nav-link active" aria-current="page">
        <i class="bi bi-grid-1x2 me-2"></i> Dashboard
      </a>
    </li>
    <li>
      <a href="#" class="nav-link text-white-50">
        <i class="bi bi-kanban me-2"></i> Kanban
      </a>
    </li>
    <li>
      <a href="#" class="nav-link text-white-50">
        <i class="bi bi-calendar3 me-2"></i> Calendar
      </a>
    </li>
    <li>
      <a href="#" class="nav-link text-white-50">
        <i class="bi bi-check2-square me-2"></i> My Tasks
      </a>
    </li>
  </ul>
</nav>
```

Reference: [Bootstrap Sidebar Example](https://getbootstrap.com/docs/5.3/examples/sidebars/)

### Topbar

Sticky top bar with page title, quick-add button, search, and user avatar.

```html
<header class="sticky-top bg-body border-bottom px-4 py-2 d-flex align-items-center gap-3">
  <h1 class="h5 mb-0 fw-semibold me-auto">Dashboard</h1>
  <div class="input-group input-group-sm" style="max-width: 260px;">
    <span class="input-group-text bg-transparent border-end-0"><i class="bi bi-search"></i></span>
    <input type="search" class="form-control border-start-0 ps-0" placeholder="Search tasks…">
  </div>
  <button class="btn btn-primary btn-sm d-flex align-items-center gap-1">
    <i class="bi bi-plus-lg"></i> New Task
  </button>
  <img src="avatar.jpg" class="rounded-circle" width="32" height="32" alt="User">
</header>
```

---

## Dashboard

The dashboard is the landing view. It surfaces high-level stats, upcoming tasks, and quick actions.

### Stat Cards

Bootstrap [Cards](https://getbootstrap.com/docs/5.3/components/card/) arranged in a responsive grid. Use `col-sm-6 col-xl-3`.

```html
<div class="row g-3 mb-4">
  <div class="col-sm-6 col-xl-3">
    <div class="card border-0 h-100" style="box-shadow: var(--app-card-shadow);">
      <div class="card-body">
        <p class="text-secondary small mb-1">Total Tasks</p>
        <h2 class="fw-bold mb-0">142</h2>
        <span class="badge bg-success-subtle text-success mt-2">+12 this week</span>
      </div>
    </div>
  </div>
  <!-- Repeat for: Completed, In Progress, Overdue -->
</div>
```

### Progress Overview

A horizontal stacked [Progress](https://getbootstrap.com/docs/5.3/components/progress/#multiple-bars) bar showing task distribution across states.

```html
<div class="progress" style="height: 8px;">
  <div class="progress-bar bg-success" style="width: 40%"></div>
  <div class="progress-bar bg-primary" style="width: 35%"></div>
  <div class="progress-bar bg-warning" style="width: 15%"></div>
  <div class="progress-bar bg-danger" style="width: 10%"></div>
</div>
```

### Today's Task List

A card with a compact list group. Tasks are checkable inline.

```html
<div class="card border-0" style="box-shadow: var(--app-card-shadow);">
  <div class="card-header bg-transparent d-flex justify-content-between align-items-center">
    <span class="fw-semibold">Today</span>
    <a href="#" class="small text-primary text-decoration-none">View all</a>
  </div>
  <ul class="list-group list-group-flush">
    <li class="list-group-item d-flex align-items-center gap-3">
      <input class="form-check-input mt-0 flex-shrink-0" type="checkbox" id="t1">
      <label class="form-check-label flex-grow-1" for="t1">Write project proposal</label>
      <span class="badge bg-warning-subtle text-warning">High</span>
    </li>
    <!-- More items -->
  </ul>
</div>
```

---

## Kanban Board

A horizontal scroll container with columns representing workflow stages.

### Columns

Each column is a card with a fixed or `min-width`. Columns scroll horizontally on overflow.

```html
<div class="d-flex gap-3 overflow-auto pb-3 align-items-start">
  <!-- Column -->
  <div class="flex-shrink-0" style="width: 300px;">
    <div class="card border-0" style="box-shadow: var(--app-card-shadow);">
      <div class="card-header bg-transparent d-flex align-items-center justify-content-between">
        <span class="fw-semibold">To Do</span>
        <span class="badge bg-secondary-subtle text-secondary">5</span>
      </div>
      <div class="card-body d-flex flex-column gap-2 p-2" id="col-todo">
        <!-- Task Cards dropped here -->
      </div>
      <div class="card-footer bg-transparent border-0">
        <button class="btn btn-sm btn-outline-secondary w-100">
          <i class="bi bi-plus"></i> Add task
        </button>
      </div>
    </div>
  </div>
  <!-- Repeat: In Progress, Review, Done -->
</div>
```

### Task Cards (Kanban)

Draggable via Sortable.js. Each card shows title, assignee avatar, priority badge, and due date.

```html
<div class="card border-0 mb-1" style="cursor: grab; box-shadow: var(--app-card-shadow);" draggable="true">
  <div class="card-body p-3">
    <div class="d-flex justify-content-between mb-2">
      <span class="badge bg-danger-subtle text-danger">High</span>
      <small class="text-secondary"><i class="bi bi-calendar2 me-1"></i>Feb 22</small>
    </div>
    <p class="mb-2 fw-medium">Redesign onboarding flow</p>
    <div class="d-flex align-items-center justify-content-between">
      <div class="d-flex">
        <img src="a1.jpg" class="rounded-circle border border-white" width="24" height="24" alt="">
        <img src="a2.jpg" class="rounded-circle border border-white ms-n1" width="24" height="24" alt="">
      </div>
      <small class="text-secondary"><i class="bi bi-paperclip me-1"></i>3</small>
    </div>
  </div>
</div>
```

### Priority Badges Palette

| Priority | Bootstrap Class |
|----------|----------------|
| High     | `bg-danger-subtle text-danger` |
| Medium   | `bg-warning-subtle text-warning` |
| Low      | `bg-success-subtle text-success` |
| None     | `bg-secondary-subtle text-secondary` |

---

## Calendar

A monthly grid view with task dots indicating scheduled items. Tasks open in an [Offcanvas](https://getbootstrap.com/docs/5.3/components/offcanvas/) detail panel on click.

### Calendar Grid Structure

```html
<div class="card border-0" style="box-shadow: var(--app-card-shadow);">
  <div class="card-header bg-transparent d-flex align-items-center justify-content-between">
    <button class="btn btn-sm btn-outline-secondary"><i class="bi bi-chevron-left"></i></button>
    <span class="fw-semibold">February 2026</span>
    <button class="btn btn-sm btn-outline-secondary"><i class="bi bi-chevron-right"></i></button>
  </div>
  <div class="card-body p-2">
    <!-- Day headers -->
    <div class="row row-cols-7 g-0 text-center mb-1">
      <div class="col"><small class="text-secondary fw-medium">Sun</small></div>
      <!-- Mon–Sat -->
    </div>
    <!-- Week rows -->
    <div class="row row-cols-7 g-0">
      <div class="col border rounded-1 p-1" style="min-height: 80px;">
        <small class="text-secondary">1</small>
        <!-- task dot indicators -->
        <span class="d-block rounded-1 px-1 mt-1 text-truncate small bg-primary-subtle text-primary">Design review</span>
      </div>
      <!-- More days -->
    </div>
  </div>
</div>
```

### Task Detail Offcanvas

Slides in from the right when a task/day is clicked.

```html
<div class="offcanvas offcanvas-end" id="taskDetail" style="width: 380px;">
  <div class="offcanvas-header border-bottom">
    <h5 class="offcanvas-title fw-semibold">Task Details</h5>
    <button class="btn-close" data-bs-dismiss="offcanvas"></button>
  </div>
  <div class="offcanvas-body">
    <!-- Title, description, assignees, due date, priority, subtasks, comments -->
  </div>
</div>
```

---

## Task Creation Modal

Uses Bootstrap [Modal](https://getbootstrap.com/docs/5.3/components/modal/) triggered by the "+ New Task" button.

```html
<div class="modal fade" id="newTaskModal" tabindex="-1">
  <div class="modal-dialog modal-dialog-centered">
    <div class="modal-content border-0" style="box-shadow: 0 20px 60px rgba(0,0,0,.15);">
      <div class="modal-header border-0 pb-0">
        <h5 class="modal-title fw-semibold">New Task</h5>
        <button class="btn-close" data-bs-dismiss="modal"></button>
      </div>
      <div class="modal-body">
        <div class="mb-3">
          <input type="text" class="form-control form-control-lg border-0 border-bottom rounded-0 px-0 fw-medium" placeholder="Task name…">
        </div>
        <textarea class="form-control border-0 px-0" rows="2" placeholder="Description (optional)"></textarea>
        <hr>
        <div class="d-flex gap-2 flex-wrap">
          <!-- Priority select -->
          <select class="form-select form-select-sm w-auto">
            <option>Priority</option>
            <option>High</option>
            <option>Medium</option>
            <option>Low</option>
          </select>
          <!-- Due date -->
          <input type="date" class="form-control form-control-sm w-auto">
          <!-- Assignee picker placeholder -->
          <button class="btn btn-sm btn-outline-secondary"><i class="bi bi-person-plus"></i></button>
        </div>
      </div>
      <div class="modal-footer border-0 pt-0">
        <button class="btn btn-secondary btn-sm" data-bs-dismiss="modal">Cancel</button>
        <button class="btn btn-primary btn-sm">Create Task</button>
      </div>
    </div>
  </div>
</div>
```

---

## Toasts & Notifications

Use Bootstrap [Toasts](https://getbootstrap.com/docs/5.3/components/toasts/) for system feedback (task created, moved, deleted). Position fixed bottom-end.

```html
<div class="toast-container position-fixed bottom-0 end-0 p-3">
  <div class="toast align-items-center border-0 bg-dark text-white" role="alert">
    <div class="d-flex">
      <div class="toast-body">
        <i class="bi bi-check-circle me-2 text-success"></i> Task moved to <strong>In Progress</strong>
      </div>
      <button class="btn-close btn-close-white me-2 m-auto" data-bs-dismiss="toast"></button>
    </div>
  </div>
</div>
```

---

## Tooltips & Popovers

Enable Bootstrap [Tooltips](https://getbootstrap.com/docs/5.3/components/tooltips/) globally for icon-only buttons and truncated labels.

```js
// Initialize all tooltips
const tooltipTriggerList = document.querySelectorAll('[data-bs-toggle="tooltip"]');
[...tooltipTriggerList].map(el => new bootstrap.Tooltip(el));
```

---

## Responsive Behavior

| Breakpoint | Sidebar | Kanban | Calendar |
|------------|---------|--------|----------|
| `< md` | Offcanvas (hamburger) | Single-column scroll | Compact list view |
| `md` | Collapsed icon-only | 2-column | Full grid |
| `lg+` | Full 240px | All columns visible | Full grid + side panel |

Use Bootstrap's `d-none d-md-flex`, `d-md-none`, and `offcanvas-md` utilities to handle breakpoint toggling without custom JS.

---

## Accessibility Notes

- All interactive elements must have visible focus styles — do not suppress `outline` globally
- Use `aria-label` on icon-only buttons: `<button aria-label="Add task"><i class="bi bi-plus-lg"></i></button>`
- Kanban drop zones should have `role="list"` and items `role="listitem"`
- Calendar days need `aria-label="February 1, 2026, 2 tasks"`
- Color is never the sole indicator of state — always pair with text or icon

---

## File Structure (Suggested)

```
/
├── index.html          # Dashboard
├── kanban.html
├── calendar.html
├── tasks.html
├── assets/
│   ├── css/
│   │   └── app.css     # Custom vars + overrides layered on Bootstrap
│   └── js/
│       └── app.js      # Bootstrap init + Sortable + interactions
└── components/         # Reusable HTML partials (if using a build tool)
    ├── sidebar.html
    ├── topbar.html
    └── task-modal.html
```

---

## Bootstrap 5.3 CDN Links

```html
<!-- CSS -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
<link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.css" rel="stylesheet">

<!-- JS Bundle (includes Popper) -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>

<!-- Sortable (Kanban drag-and-drop) -->
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.2/Sortable.min.js"></script>
```

---

*Last updated: February 2026*
