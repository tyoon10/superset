---
name: testing-superset-local
description: How to run and smoke-test the Superset backend locally (no frontend build) — DB init, admin login, API checks
---

# Testing Superset backend locally without building the frontend

Building `superset-frontend` (npm ci + webpack) is very heavy (needs Node ^24, lots of RAM/time) and is usually unnecessary for backend/dependency changes. The Flask backend can be fully exercised over HTTP.

## Setup
1. Use a Python 3.11 venv with `requirements/development.txt` plus `pip install -e .` (system deps needed: pkg-config, libmysqlclient-dev, libpq-dev, libsasl2-dev, libldap2-dev).
2. The app refuses to start with the default SECRET_KEY — export `SUPERSET_SECRET_KEY=$(openssl rand -hex 24)` for every command/process.
3. Initialize the metadata DB (SQLite by default at `~/.superset/superset.db`):
   ```bash
   superset db upgrade
   superset fab create-admin --username admin --firstname Admin --lastname User --email admin@example.com --password admin
   superset init
   ```
4. Start the dev server: `superset run -p 8088 --with-threads` (backgrounded; logs are noisy DEBUG watchdog lines — filter for `HTTP/1.1" 5` and `Traceback` to spot real errors).

## Key facts
- `GET /health` → 200 `OK`; good readiness probe.
- Without built assets (`superset/static/assets` missing), `/login/` and `/superset/welcome/` return 200 HTML shells but render BLANK in a browser (React bootstrap). Browser-based UI testing is not possible without `cd superset-frontend && npm ci && npm run build` — prefer HTTP-level testing instead.
- Session login via HTTP: GET `/login/`, extract `csrf_token` (regex `name="csrf_token" ... value="..."` in the HTML), POST `/login/` with `username`, `password`, `csrf_token` using the same cookie jar → 302 to `/` on success, 302 back to `/login/?next=/` on bad credentials.
- After login, `GET /api/v1/me/` returns 200 with the user JSON (401 `Missing Authorization Header` when unauthenticated).
- Good no-error smoke endpoints: `/api/v1/dashboard/`, `/api/v1/chart/`, `/api/v1/database/`, `/api/v1/saved_query/` — all return 200 JSON with a `count` key on a fresh DB.
- On Flask >= 3.1 (CVE-2026-27205 fix), any response where the session was accessed carries `Vary: ... Cookie` (e.g. `Vary: Accept-Encoding, Cookie` on `/health`, `/login/`, API responses).

## Devin Secrets Needed
None — everything runs locally with a generated `SUPERSET_SECRET_KEY`.
