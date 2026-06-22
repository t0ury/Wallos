# Wallos

Self-hosted PHP 8.3 subscription tracker: SQLite3 database, Nginx, no frontend build step, deployed as a Docker image.

## Coding standards

All code MUST follow `docs/standards/ENGINEERING-GUIDE.portable.md` (git submodule). Most-violated rule: **validate all user input at the boundary; trust internal code paths.**

## Stack / commands

**Run:**
```bash
docker compose up          # http://localhost:8282
```

**Migrate DB manually:**
```bash
php endpoints/db/migrate.php
```

**Run a cronjob manually:**
```bash
php endpoints/cronjobs/updatenextpayment.php
```

No build step, no test suite, no linter. PHP files are served directly.

## Architecture

**Two endpoint families — do not mix their patterns:**

| Family | Path | Auth | DB path |
|---|---|---|---|
| Internal AJAX (browser UI) | `endpoints/**/*.php` | POST + session + CSRF via `includes/validate_endpoint.php` | `../../db/wallos.db` via `includes/connect_endpoint.php` |
| External REST API | `api/**/*.php` | API token via `includes/validate_endpoint_admin.php` | same |

Page-level files (`index.php`, `stats.php`, …) are server-rendered and must start with `require_once 'includes/header.php'`.

**Database:** SQLite3 at `db/wallos.db`. Schema changes go in a new sequentially-numbered file in `migrations/` (e.g. `000029.php`) — never modify existing migration files. Migrations are idempotent and auto-run at Docker startup.

**Key shared includes:**

- `includes/inputvalidation.php` — `validate()`: trim + stripslashes + htmlspecialchars. Call on every value from `$_POST`/`$_GET`.
- `includes/ssrf_helper.php` — must wrap every outbound HTTP request (logo fetch, webhook, etc.) to block private/CGNAT IPs.
- `libs/csrf.php` — `generate_csrf_token()` / `verify_csrf_token()`. Every internal POST endpoint already pulls this via `validate_endpoint.php`; do not bypass it.
- `includes/getsettings.php` — loads per-user settings into `$settings[]`.

**i18n:** PHP strings in `includes/i18n/{lang}.php`; JS strings in `scripts/i18n/{lang}.js`. Both files required for a complete translation.

## Security invariants — never break these

1. All `$_POST`/`$_GET` values passed to SQL must use prepared statements (`$db->prepare()`).
2. User-supplied URLs must be validated through `ssrf_helper.php` before any `curl_exec`.
3. Internal endpoints must include `validate_endpoint.php` (enforces POST + CSRF + session).
4. Never expose the raw SQLite error to the client response.
