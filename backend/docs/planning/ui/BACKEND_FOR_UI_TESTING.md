# Backend For UI Testing

Use this runbook when a frontend developer needs a GroupScout backend available while building or testing the operator UI.

The current backend is ready for health checks, pipeline triggers, metrics, and legacy raw audit lookup. The UI-specific lead list/detail, outreach, pipeline, stats, system, auth/session, and alert compatibility endpoints are planning contracts, not live routes on backend `main`; implementation or merge work is tracked by `groupscout-site-eqm`.

## Quick Start: Local Backend

This is the fastest path for UI work because it avoids the full Docker stack and does not pull Ollama models.

```bash
DATABASE_URL=groupscout-ui-dev.db \
API_TOKEN=dev-token \
ADMIN_AUTH_ENABLED=true \
OLLAMA_ENABLED=false \
ENRICHMENT_ENABLED=false \
go run cmd/server/main.go
```

The server listens on `http://localhost:8080` unless `PORT` is set.

Verify it:

```bash
curl -i http://localhost:8080/health
```

The planned admin setup-token flow is not available on the current backend source snapshot. Until the auth/session routes land, use mocked UI auth, a dev proxy, or server-to-server `API_TOKEN` calls outside browser JavaScript.

Planned auth smoke, once implemented:

```bash
curl -i http://localhost:8080/api/auth/status
SETUP_TOKEN="$(cat data/admin-setup-token)"
curl -i -c /tmp/groupscout-admin.cookies \
  -H "Content-Type: application/json" \
  -d "{\"token\":\"${SETUP_TOKEN}\"}" \
  http://localhost:8080/api/auth/login
curl -i -b /tmp/groupscout-admin.cookies http://localhost:8080/api/auth/me
```

Successful login should set an HttpOnly `groupscout_session` cookie and return a `session_token` for non-browser smoke clients. File-backed setup tokens should rotate after login, so read the token file again if another setup login is needed. Env-backed `ADMIN_SETUP_TOKEN` values cannot rotate automatically.

Logout revokes the current session and expires the cookie:

```bash
curl -i -b /tmp/groupscout-admin.cookies -c /tmp/groupscout-admin.cookies \
  -X POST http://localhost:8080/api/auth/logout
```

Expected result is `200 OK` with JSON similar to:

```json
{"database":"ok","ollama":"unavailable","status":"ok"}
```

## Postgres-Backed Local Backend

Use this when the UI needs behavior closer to production persistence, migrations, or pgvector.

```bash
docker compose up -d postgres
docker compose ps
```

Then start the Go server against the local Postgres port:

```bash
DATABASE_URL='postgres://groupscout:groupscout@localhost:5432/groupscout?sslmode=disable' \
API_TOKEN=dev-token \
OLLAMA_ENABLED=false \
ENRICHMENT_ENABLED=false \
go run cmd/server/main.go
```

Verify:

```bash
curl -i http://localhost:8080/health
```

## Full Docker Stack

Use this only when UI testing also needs the containerized app, n8n, Grafana, Prometheus, Loki, or the Ollama-backed enrichment path.

```bash
docker compose up -d groupscout
docker compose ps
docker compose logs -f groupscout
```

Service URLs:

| Service | URL |
|---|---|
| GroupScout API | `http://localhost:8080` |
| n8n | `http://localhost:5678` |
| Grafana | `http://localhost:3000` |
| Prometheus | `http://localhost:9090` |

The `groupscout` compose service depends on Postgres, Ollama, and `ollama-init`. First boot can take several minutes because `ollama-init` pulls models.

For the Docker backend, read the current setup token from the app container:

```bash
docker exec groupscout_app sh -lc 'cat data/admin-setup-token'
```

The browser login route is served by the UI container at:

```txt
http://localhost:3001/admin/login
```

## Backend Plus UI Docker Smoke

For a single backend-plus-frontend Docker smoke path, see [BACKEND_FRONTEND_DOCKER_E2E.md](./BACKEND_FRONTEND_DOCKER_E2E.md).

Keep this file focused on starting a backend for UI work. The Docker E2E runbook owns the Compose override, UI dev-server, D4 production proxy, health curls, and troubleshooting expectations.

## API Calls Useful During UI Work

Health check:

```bash
curl -i http://localhost:8080/health
```

Pipeline trigger:

```bash
curl -i -X POST http://localhost:8080/run \
  -H "Authorization: Bearer dev-token" \
  -H "Content-Type: application/json" \
  -d '{"bcbid_raw_input": ""}'
```

Single raw project/event ingest:

```bash
curl -i -X POST http://localhost:8080/ingest \
  -H "Authorization: Bearer dev-token" \
  -H "Content-Type: application/json" \
  -d '{"source":"manual","title":"Richmond warehouse infrastructure","location":"Richmond BC","project_value":12000000,"description":"Civil warehouse expansion with likely travelling crews."}'
```

Postgres integration follow-up for the enrichment raw-input FK path is tracked by `groupscout-site-wda`; the handler and single-item enrichment unit suites pass.

Metrics:

```bash
curl -i http://localhost:8080/metrics
```

Raw audit payload for a known lead:

```bash
curl -i http://localhost:8080/leads/LEAD_ID/raw
```

## Frontend Wiring Notes

Do not put `API_TOKEN` in browser JavaScript. It is an automation token for server-to-server calls such as n8n. A browser UI should use a same-origin session boundary, a local dev proxy, or mocked API data until the UI auth layer exists.

If the UI dev server runs on a different port, prefer a dev-server proxy that forwards API calls to `http://localhost:8080`. The Go server does not currently implement a browser CORS policy for the future UI.

Example Vite-style proxy target:

```text
/health -> http://localhost:8080/health
/metrics -> http://localhost:8080/metrics
/api/* -> http://localhost:8080/api/*
```

The `/api/*` routes above are the planned UI contract. They stay separate from the automation endpoints so browser response shapes and session boundaries can evolve without exposing `API_TOKEN`.

## Current UI Contract Status

Implemented today:

| Endpoint | UI testing use |
|---|---|
| `GET /health` | Backend reachability and database readiness. |
| `GET /metrics` | Observability smoke check; not intended as direct UI data. |
| `POST /run` | Manual pipeline trigger behind Bearer auth. |
| `POST /digest` | Manual digest trigger behind Bearer auth. |
| `POST /ingest` | External raw project/event ingest through `EnrichOne()` behind Bearer auth. |
| `POST /n8n/webhook` | External pre-enriched lead direct insert behind Bearer auth. |

Planned UI contract, not live on backend `main`:

| Endpoint | UI testing use |
|---|---|
| `GET /api/auth/status`, `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me` | Admin setup-token/session flow. |
| `GET /api/leads` | Lead inbox with filters and pagination. |
| `GET /api/leads/{id}` | Lead detail. |
| `PATCH /api/leads/{id}` | Planned status, owner, notes, snooze, and workflow actions. Audit-safe corrections remain unsupported. |
| `GET /api/leads/{id}/raw` | Authenticated UI alias for raw audit evidence. |
| `GET/POST /api/leads/{id}/outreach` | Outreach activity history and logging. |
| `GET/POST /api/pipeline/runs` | Run history and async pipeline controls. |
| `GET /api/stats` | UI summaries by status, source, score band, owner, and week. |
| `GET /api/system` | UI-friendly system health summary. |
| `GET /api/alerts` | Read-only alert-console compatibility endpoint. |

## Troubleshooting

| Symptom | Check |
|---|---|
| `connection refused` on `localhost:8080` | Confirm the Go process is still running or `docker compose ps` shows `groupscout` up. |
| `401 Unauthorized` on `POST /run` | Match the Bearer value to `API_TOKEN`; for this runbook use `Bearer dev-token`. |
| `/health` returns database error | Check `DATABASE_URL`; for local SQLite use a writable `.db` path, for Postgres verify `docker compose ps postgres`. |
| Docker startup is slow | `ollama-init` is likely pulling models. Use the local backend path for UI-only work. |
| Browser requests fail from a UI dev server | Add a dev proxy or same-origin wrapper; do not rely on CORS being enabled in the Go server. |
| D4 `/api/system` or `/api/alerts` returns `404` | Expected on backend `main` until protected UI API routes land; a `502` from the intentionally bad proxy target is the proxy-failure signal. |
