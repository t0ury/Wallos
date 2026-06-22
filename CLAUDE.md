# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What Wallos Is

Wallos is a self-hosted PHP web application for tracking personal and household subscriptions. It uses SQLite3 for storage, Nginx as the web server, and is distributed as a Docker image (`bellamy/wallos`).

## Running Locally

**Docker (recommended):**
```bash
docker compose up
# App is available at http://localhost:8282
```

**Manual DB migration** (after pulling changes):
```bash
php /var/www/html/endpoints/db/migrate.php
# or via browser: http://localhost:8282/endpoints/db/migrate.php
```

**Run a specific cronjob manually:**
```bash
php endpoints/cronjobs/updatenextpayment.php
php endpoints/cronjobs/updateexchange.php
```

There is no build step, test suite, or linter configured in this repository. Changes take effect immediately when files are served by PHP.

## Architecture

### Request Flow

There are two distinct families of PHP endpoints:

**1. Internal AJAX endpoints (`endpoints/`)**
Used by the browser UI. All requests must be `POST` with a valid session and CSRF token.
- Session + CSRF validation is provided by including `includes/validate_endpoint.php` at the top.
- Database connection is from `includes/connect_endpoint.php` (resolves `db/wallos.db` relative to `../../`).
- Organized by resource: `endpoints/subscription/`, `endpoints/subscriptions/`, `endpoints/cronjobs/`, `endpoints/db/`, etc.

**2. External REST API (`api/`)**
Token-based API for programmatic access. Uses `includes/validate_endpoint_admin.php`.
- Organized by resource: `api/subscriptions/`, `api/categories/`, `api/users/`, etc.

**Page-level PHP files** (`index.php`, `stats.php`, `calendar.php`, etc.) are server-rendered. They all start with `require_once 'includes/header.php'`, which handles session, auth redirect, i18n, settings, and CSRF token generation.

### Database

SQLite3 file at `db/wallos.db`. The empty schema template is `db/wallos.empty.db`.

**Migrations** live in `migrations/` as sequentially-numbered PHP files (`000001.php`, `000002.php`, …). They are tracked in a `migrations` table and run idempotently. Add new migrations by creating the next numbered file — never modify existing ones.

Migrations auto-run on Docker startup via `startup.sh`. For baremetal, run `endpoints/db/migrate.php` manually after updates.

### Shared Includes (`includes/`)

| File | Purpose |
|---|---|
| `connect.php` | SQLite3 connection for page-level files (relative path `db/wallos.db`) |
| `connect_endpoint.php` | SQLite3 connection for `endpoints/` files (relative path `../../db/wallos.db`) |
| `header.php` | Bootstrap for all authenticated pages: DB, session, auth check, i18n, settings, CSRF |
| `validate_endpoint.php` | Enforces POST method + valid CSRF token + active session for internal endpoints |
| `validate_endpoint_admin.php` | API token validation for external REST API |
| `getsettings.php` | Loads per-user settings from the `settings` table into `$settings[]` |
| `inputvalidation.php` | `validate()` helper: trim + stripslashes + htmlspecialchars |
| `ssrf_helper.php` | IP validation for outbound HTTP requests (logo fetching, webhooks) |
| `csrf.php` (in `libs/`) | `generate_csrf_token()` and `verify_csrf_token()` |
| `run_migrations.php` | Migration runner logic (expects `$db` to be set by caller) |

### JavaScript & CSS

- JS files in `scripts/` are per-page (`subscriptions.js`, `stats.js`, etc.) plus shared utilities (`common.js`, `theme.js`).
- i18n strings for JS live in `scripts/i18n/{lang}.js`.
- Styles in `styles/`. No CSS preprocessor — plain CSS.
- No frontend build toolchain (no npm, webpack, etc.).

### Cronjobs

Scheduled tasks in `endpoints/cronjobs/`. The `cronjobs` file at repo root is the crontab loaded into the Docker container. For baremetal, install those entries via `crontab -e`.

## Internationalization

To add a new language:
1. Add the language code to `includes/i18n/languages.php`.
2. Copy `includes/i18n/en.php` → `includes/i18n/{lang}.php` and translate all values.
3. Copy `scripts/i18n/en.js` → `scripts/i18n/{lang}.js` and translate all values.
4. Incomplete translations are not accepted.

## Engineering Standards

This project follows the engineering principles in [`docs/standards/ENGINEERING-GUIDE.portable.md`](./docs/standards/ENGINEERING-GUIDE.portable.md) (included via git submodule). Key points:

- Commit messages use Conventional Commits (`feat:`, `fix:`, `chore:`, etc.).
- PRs should be focused: one feature or bug fix per PR.
- Input from users/external sources must be sanitized via `validate()` from `inputvalidation.php`. Trust internal PHP code paths.
- All outbound HTTP requests must go through SSRF validation (`ssrf_helper.php`) before fetching.

## Security Patterns

- CSRF tokens are generated per-session and must accompany every internal POST endpoint call (either as `$_POST['csrf_token']` or `HTTP_X_CSRF_TOKEN` header).
- The first registered user (userId = 1) is the admin.
- Logo URL fetching validates IPs against private/reserved ranges and CGNAT before following redirects.
