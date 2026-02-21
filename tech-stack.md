# Tech Stack: Todo List Tracker Application

## Overview

This document describes the complete technology stack for the **Todo List Tracker** web application. It is intended to guide Claude Code in scaffolding, building, and deploying the application correctly. Follow the versions, conventions, and integration notes precisely.

---

## Infrastructure: LAMP Stack

| Layer       | Technology          | Version       |
|-------------|---------------------|---------------|
| OS          | Ubuntu              | 24.04 LTS     |
| Web Server  | Apache              | 2.4.x (latest available via apt) |
| Database    | MariaDB             | 10.11.x LTS (latest available via apt) |
| Language    | PHP                 | 8.3.x (latest available via apt) |

### Apache Configuration Notes
- Enable `mod_rewrite` for clean URL routing: `a2enmod rewrite`
- Enable `mod_headers` for security headers: `a2enmod headers`
- Set `AllowOverride All` in the VirtualHost config to support `.htaccess` files
- Document root: `/var/www/html/todo/public/`
- Use a `.htaccess` file to route all requests through a single front controller (`index.php`)

### MariaDB Configuration Notes
- Database name: `todo_app`
- Create a dedicated MariaDB user with limited privileges (no root access in app)
- Use `utf8mb4` character set and `utf8mb4_unicode_ci` collation on all tables
- Store credentials in a `.env` file outside the document root; never hardcode them
- Install via `apt`: `sudo apt install mariadb-server` — do **not** install `mysql-server`
- Run `sudo mysql_secure_installation` after install to harden the server

### PHP Configuration Notes
- Required extensions: `pdo`, `pdo_mysql`, `mbstring`, `json`, `session` — `pdo_mysql` is compatible with MariaDB; no separate driver needed
- Enable short open tags off; use full `<?php` tags
- Set `display_errors = Off` in production; log errors to file
- Use PHP sessions for user state management (if auth is added later)

---

## Project Directory Structure

```
/var/www/html/todo/
├── public/                  # Document root (Apache points here)
│   ├── index.php            # Front controller
│   ├── .htaccess            # URL rewriting rules
│   └── assets/
│       ├── css/
│       │   └── app.css      # Custom styles (loaded after Bootstrap)
│       └── js/
│           └── app.js       # Custom JS (loaded after all libraries)
├── src/
│   ├── Controllers/         # PHP controller classes
│   ├── Models/              # PHP model/data-access classes
│   └── Views/               # PHP HTML templates (partials + layouts)
├── config/
│   └── database.php         # DB connection using PDO
├── .env                     # Environment variables (NOT committed to git)
├── .env.example             # Template for .env
└── tech-stack.md            # This file
```

---

## Frontend Libraries

All libraries are loaded via CDN in the correct order. Do **not** use npm/bundlers for this project — this is a traditional LAMP stack with no build step.

### 1. Bootstrap 5.3
- **Version:** 5.3.x (latest stable)
- **Source:** Load CSS and JS from the official Bootstrap CDN as documented at https://getbootstrap.com/
- **Theme:** Use the default Bootstrap theme. Reference https://getbootstrap.com/ for component markup and class conventions.
- **JavaScript bundle:** Use the full `bootstrap.bundle.min.js` (includes Popper.js)

**CDN Links (load in `<head>` and before `</body>`):**
```html
<!-- Bootstrap CSS — in <head> -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH" crossorigin="anonymous">

<!-- Bootstrap JS bundle (includes Popper) — before </body> -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" integrity="sha384-YvpcrYf0tY3lHB60NNkmXc4s9bIOgUxi8T/jzmVOFCBpA2JBEuSHQg4sLkKGEo=" crossorigin="anonymous"></script>
```

### 2. jQuery
- **Version:** 3.7.x (latest stable — Bootstrap 5 does not require jQuery, but it is included per project requirements)
- **Load order:** jQuery must load **before** Bootstrap JS and before HTMX/Alpine

**CDN Link:**
```html
<!-- jQuery — before Bootstrap JS, before </body> -->
<script src="https://cdn.jsdelivr.net/npm/jquery@3.7.1/dist/jquery.min.js" integrity="sha256-/JqT3SQfawRcv/BIHPThkBvs0OEvtFFmqPF/lYI/Cxo=" crossorigin="anonymous"></script>
```

### 3. HTMX 2.0.8
- **Version:** 2.0.8 (current stable release of the 2.x line)
- **Purpose:** Handle all AJAX requests declaratively in HTML — adding, editing, deleting, and filtering todos without full page reloads
- **Load order:** Load **after** jQuery, **before** Alpine.js
- **Key attributes used in this project:** `hx-get`, `hx-post`, `hx-put`, `hx-delete`, `hx-target`, `hx-swap`, `hx-trigger`, `hx-confirm`, `hx-indicator`
- **Swap strategy:** Default to `hx-swap="outerHTML"` for todo items; use `hx-swap="innerHTML"` for list containers
- **Server responses:** PHP endpoints must return HTML fragments (not JSON) for HTMX to swap into the DOM

**CDN Link:**
```html
<!-- HTMX — after jQuery, before Alpine, before </body> -->
<script src="https://cdn.jsdelivr.net/npm/htmx.org@2.0.8/dist/htmx.min.js" integrity="sha384-/TgkGk7p307TH7EXJDuUlgG3Ce1UVolAOFopFekQkkXihi5u/6OCvVKyz1W+idaz" crossorigin="anonymous"></script>
```

> **HTMX 2.x Breaking Change Note:** HTMX 2.x no longer includes extensions in the core bundle. If extensions are needed (e.g., `response-targets`), load them separately from https://extensions.htmx.org.

### 4. Alpine.js 3.15.8
- **Version:** 3.15.8 (current latest stable)
- **Purpose:** Handle local UI interactivity — toggling edit mode, form validation feedback, modal state, inline confirmation dialogs
- **Load order:** Load **after** HTMX, **before** `</body>`. Must use the `defer` attribute.
- **Key directives used:** `x-data`, `x-show`, `x-model`, `x-on` / `@click`, `x-bind`, `x-transition`, `x-ref`
- **Integration with HTMX:** Alpine manages client-side UI state; HTMX handles server communication. Avoid overlap — do not use Alpine to make fetch/XHR calls; do not use HTMX for purely visual toggles.

**CDN Link:**
```html
<!-- Alpine.js — last JS library, before </body>, MUST have defer -->
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.15.8/dist/cdn.min.js"></script>
```

### Script Load Order Summary

Inside `<head>`:
1. Bootstrap CSS

Before `</body>` — in this exact order:
1. jQuery
2. Bootstrap JS bundle
3. HTMX
4. Alpine.js *(with `defer`)*
5. `assets/js/app.js` *(custom JS, if any)*

---

## PHP Backend Conventions

- **Architecture:** Simple MVC-lite pattern using plain PHP (no framework)
- **Routing:** Single front controller (`public/index.php`) dispatches requests based on `$_GET['route']` or URL path parsed from `$_SERVER['REQUEST_URI']`
- **Database access:** PDO with prepared statements only — use DSN prefix `mysql:` (PDO uses this for MariaDB too); no raw string interpolation in queries
- **HTMX responses:** Controllers detect `HX-Request` header (`$_SERVER['HTTP_HX_REQUEST']`) to decide whether to return a full page or just an HTML partial
- **HTTP methods:** Use `$_SERVER['REQUEST_METHOD']` to handle GET, POST, PUT/PATCH, DELETE — Apache must allow all methods
- **CSRF protection:** Generate and validate a CSRF token stored in `$_SESSION` for all mutating requests (POST, PUT, DELETE)

---

## Database Schema

### `todos` Table
```sql
CREATE TABLE todos (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    description TEXT,
    is_complete TINYINT(1) NOT NULL DEFAULT 0,
    priority    ENUM('low', 'medium', 'high') NOT NULL DEFAULT 'medium',
    due_date    DATE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## Application Features

The following features must be implemented:

1. **List all todos** — display todos in a Bootstrap card/list-group layout, filterable by status (all / active / complete) and sortable by priority or due date
2. **Add a todo** — inline form using HTMX POST; new item prepended to list without page reload
3. **Edit a todo** — clicking edit swaps the display row for an inline edit form (Alpine.js controls toggle; HTMX submits the PUT)
4. **Delete a todo** — HTMX DELETE with `hx-confirm` prompt; removes item from DOM on success
5. **Toggle complete** — checkbox triggers HTMX PATCH; row style updates to show completion state
6. **Filter/search** — Alpine.js or HTMX-powered filter by status and priority

---

## Security Checklist

- [ ] All SQL queries use PDO prepared statements
- [ ] CSRF tokens on all mutating forms
- [ ] `htmlspecialchars()` used on all output
- [ ] `.env` file is outside document root and blocked by Apache (`Deny from all`)
- [ ] HTTP security headers set in Apache config or `.htaccess` (`X-Frame-Options`, `X-Content-Type-Options`, `Content-Security-Policy`)
- [ ] `display_errors` is off in production PHP config
- [ ] MariaDB user has only SELECT, INSERT, UPDATE, DELETE privileges on `todo_app` database (no DROP, no GRANT)

---

## Deployment Notes

- This application is deployed directly to the LAMP server — no CI/CD pipeline, no containers
- Use `git pull` to deploy updates to `/var/www/html/todo/`
- Apache VirtualHost config file location: `/etc/apache2/sites-available/todo.conf`
- Enable site with: `a2ensite todo.conf && systemctl reload apache2`
- MySQL setup script: `config/setup.sql` (creates database, MariaDB user, and tables)
- File permissions: web files owned by `www-data`; `.env` permissions set to `640`

---

## Reference Links

| Resource | URL |
|---|---|
| Bootstrap 5.3 Docs | https://getbootstrap.com/ |
| HTMX 2.x Docs | https://htmx.org/docs/ |
| HTMX Extensions | https://extensions.htmx.org |
| Alpine.js 3.x Docs | https://alpinejs.dev/ |
| jQuery 3.x Docs | https://api.jquery.com/ |
| PHP 8.3 Docs | https://www.php.net/docs.php |
| PDO Reference | https://www.php.net/manual/en/book.pdo.php |
