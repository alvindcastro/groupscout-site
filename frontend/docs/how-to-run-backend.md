# How To Run The Backend

This document is maintained in the coordinator repo at:

```sh
/mnt/c/Users/alvin/groupscout-site/frontend/docs/how-to-run-backend.md
```

Run backend commands from the backend source repo:

```sh
/mnt/c/Users/alvin/GolandProjects/groupscout
```

Use these notes when you need API data for UI contract work or local integration checks. Long-lived markdown docs now live under `/mnt/c/Users/alvin/groupscout-site/backend` and `/mnt/c/Users/alvin/groupscout-site/frontend`; the backend source repo remains the source of truth for current code, config, Compose files, and Makefile targets when examples disagree.

## Prerequisites

- Go `1.26+`
- Docker and Docker Compose for Postgres, n8n, monitoring, and the full stack
- `pdftotext` for PDF permit scraping when running collectors locally
- A backend `.env` copied from `.env.example`

Minimum backend `.env` values called out by the backend docs:

```env
CLAUDE_API_KEY=sk-ant-...
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/XXX/YYY/ZZZ
API_TOKEN=your_secure_token_here
DATABASE_URL=groupscout.db
```

Note: backend email-provider docs have had `SENDGRID_API_KEY` versus `RESEND_API_KEY` drift. Prefer the backend repo's current `.env.example` and `config/config.go` when configuring email.

For Postgres local development:

```env
DATABASE_URL=postgres://groupscout:groupscout@localhost:5432/groupscout
```

## Local Go Server

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
go mod download
go run cmd/server/main.go
```

Expected API base:

```txt
http://localhost:8080
```

Useful endpoints:

- `GET /health`
- `POST /run`
- `POST /digest?to=email@example.com`
- `POST /ingest` for raw or lightly normalized project/event items that should run through GroupScout enrichment
- `POST /n8n/webhook` for pre-enriched lead-shaped direct inserts

Most write or trigger endpoints expect:

```txt
Authorization: Bearer YOUR_API_TOKEN
```

## One-Shot Pipeline

Run one collector/enrichment/notification pass and exit:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
go run cmd/server/main.go --run-once
```

Equivalent Makefile target:

```sh
make run-once
```

## Postgres-Only Setup

Use this when the backend should run locally but storage should match the Postgres path:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
docker compose up postgres -d
docker compose ps
```

For the current testing session on 2026-05-09, this command confirmed `groupscout_postgres` is already running and healthy on `localhost:5432`.

Set:

```env
DATABASE_URL=postgres://groupscout:groupscout@localhost:5432/groupscout
```

Then run:

```sh
go run cmd/server/main.go --run-once
```

Migrations run automatically on first boot.

## Full Docker Stack

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
docker compose up -d
```

Services exposed by the current compose file:

| Service | URL / Port | Purpose |
|---|---:|---|
| GroupScout API | `http://localhost:8080` | Lead generation server |
| Alertd | `http://localhost:8081` | Airport disruption monitor |
| Postgres | `localhost:5432` | Primary database with pgvector |
| n8n | `http://localhost:5678` | Workflow scheduler |
| Prometheus | `http://localhost:9090` | Metrics |
| Grafana | `http://localhost:3000` | Dashboards and logs |
| Loki | `http://localhost:3100` | Log aggregation |

Important: the compose service is currently named `groupscout`, with container name `groupscout_app`. Some backend docs still mention `app` in log commands. Prefer:

```sh
docker compose logs -f groupscout
docker compose logs groupscout --tail=50
```

If that fails, inspect service names:

```sh
docker compose ps
```

## Run Backend With UI Docker

From the UI repo, choose the current D3 development product server or D4 production static/proxy smoke path in [Docker Runtime Matrix](./docker-runtime-matrix.md). That matrix owns the merged Compose command, `-p groupscout` network convention, D4 production-container command, expected `/api/*` smoke statuses, and cleanup steps.

Expected endpoints:

- Backend API: `http://localhost:8080`
- Backend health: `http://localhost:8080/health`
- Backend Grafana, when the full stack is started: `http://localhost:3000`
- UI product dev server health: `http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/healthz`
- UI admin login: `http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/admin/login`

The UI service in `compose.dev.yml` is now the Phase 13 product dev server. It serves generated `web/dist` assets, healthchecks `/healthz`, and keeps `http://groupscout:8080` backend discovery server-side.

Current smoke note: `GET http://localhost:8080/health` returns `200` with `"database":"ok"`. It may also report `"ollama":"unavailable"` even when the `groupscout_ollama` container is healthy. That is acceptable for UI/API smoke and Postgres tests, but resolve it before testing LLM enrichment or model-dependent pipeline behavior.

When testing the admin flow through the UI service, `/admin/login` should show a compact floating setup-token window. If it shows a wide embedded panel, rebuild from the UI repo with `npm run build` and restart `groupscout-ui` with `--build`.

For a same-origin UI runtime against the backend container, use the D4 production image command in [Docker Runtime Matrix](./docker-runtime-matrix.md).

## Alertd

Run the airport disruption monitor locally:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
go run cmd/alertd/main.go
```

Alertd listens on `localhost:8081` by default and needs `config/airports.yaml`.

Manual slash-command simulation:

```sh
curl -X POST -d "command=/inventory&text=34" http://localhost:8081/slack/inventory
```

## Health And Trigger Checks

```sh
curl -i http://localhost:8080/health
```

Admin auth status, setup-token login, token rotation, logout, and stale asset recovery are covered in [Admin Token Flow](./admin-token-flow.md).

```sh
curl -i -X POST \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  http://localhost:8080/run
```

For container logs after a run:

```sh
docker compose logs groupscout --tail=50
```

## Centralized Backend Docs

Primary backend references:

- `/mnt/c/Users/alvin/groupscout-site/backend/README.md`
- `/mnt/c/Users/alvin/groupscout-site/backend/docs/DEVELOPER.md`
- `/mnt/c/Users/alvin/groupscout-site/backend/docs/guides/SETUP.md`
- `/mnt/c/Users/alvin/groupscout-site/backend/docs/guides/DOCKER.md`
- `/mnt/c/Users/alvin/groupscout-site/backend/docs/guides/TESTING.md`
- `/mnt/c/Users/alvin/groupscout-site/backend/docs/guides/TROUBLESHOOTING.md`
