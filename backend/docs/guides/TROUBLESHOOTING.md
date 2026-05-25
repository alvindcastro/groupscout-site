# Troubleshooting — Missing Leads & Pipeline Gaps

This guide helps you diagnose why the GroupScout pipeline may return "0 new leads" or fewer leads than expected.

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/backend/docs/guides/TROUBLESHOOTING.md`. Run backend diagnostics from `/mnt/c/Users/alvin/GolandProjects/groupscout`; run UI diagnostics from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

## Operator Quick Check

Run from `/mnt/c/Users/alvin/GolandProjects/groupscout`:

```bash
docker compose ps
curl -i --max-time 5 http://localhost:8080/health
docker exec groupscout_postgres psql -U groupscout -d groupscout -c "SELECT status, count(*) FROM leads GROUP BY status ORDER BY status;"
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
```

Expected:

- Backend health is HTTP `200` and includes `"database":"ok"`.
- `ollama` may be `degraded` or `unavailable` for UI/API smoke, but must be healthy before claiming LLM enrichment is working.
- Postgres query succeeds. Zero `new` leads is not automatically a failure; check filter and dedup logs below.
- n8n reaches the backend over `http://groupscout:8080`.

If a check fails:

```bash
docker compose logs --tail=100 groupscout postgres n8n ollama
```

## 🔍 Core Pipeline Flow

GroupScout filters data at three distinct stages. Understanding these is key to troubleshooting:

1.  **Collector-level Filter**: Each source (Richmond, Delta, BCBid, etc.) has its own relevance rules. For example, Richmond building permits are filtered by a **minimum value** (default $500,000) and **commercial sub-types** (e.g., "hotel", "warehouse", "apartment"). Anything else is skipped before it even reaches the main pipeline.
2.  **Database Deduplication**: Before processing a lead, the system checks the `raw_projects` table for a matching SHA256 hash (based on title, location, and date). If it's already in the database, it's skipped.
3.  **Pre-scoring (Enrichment Threshold)**: The Go pre-scoring engine calculates a score based on location (Richmond/YVR gets +2) and keywords. If the score is below the `ENRICHMENT_THRESHOLD` (default: 1), it is marked as `skipped` and **will not** be enriched by AI or sent to Slack.

---

## 🛠 Troubleshooting Steps

### 1. Check the Logs
If you run `go run cmd/server/main.go --run-once`, look for these log markers:

For Docker runs:

```bash
docker compose logs groupscout --tail=200 | rg "filtering complete|skipping duplicate|skipping enrichment|sent leads|no new leads|slack notify|error"
```

- `parsed records from PDF count=102`: How many raw entries were found before filtering.
- `filtering complete source=richmond passed=5 skipped_low_value=90 skipped_residential=7`: Detailed breakdown of why permits were excluded.
- `skipping duplicate`: The permit was already processed in a previous run.
- `skipping enrichment: low score score=0`: The project was recorded but wasn't interesting enough to justify AI costs.
- `sent leads to Slack count=N`: the run produced and delivered leads.
- `no new leads to notify`: the run completed, but there were no fresh `new` leads to send.

Actions by marker:

| Marker | What to check |
|---|---|
| `skipped_low_value` | Lower `MIN_PERMIT_VALUE_CAD` for testing or accept that small projects are filtered. |
| `skipped_residential` | Expected for residential permits; adjust collector rules only if residential work should become eligible. |
| `skipping duplicate` | The source item was already processed. Inspect `raw_projects` or clear test data only when deliberately replaying. |
| `skipping enrichment: low score` | Lower `ENRICHMENT_THRESHOLD` for testing or improve scoring rules. |
| `slack notify` | Check `SLACK_WEBHOOK_URL` and Slack webhook status. |
| collector or `pdftotext` error | Rebuild with `docker compose up -d --build` and confirm `poppler-utils`/`pdftotext` exists in the image. |

Guaranteed cadence runs add a `delivery_status` field to `/run` JSON:

| Status | What to check |
|---|---|
| `sent` | One eligible lead was delivered and recorded in `lead_deliveries`. |
| `duplicate` | The same `idempotency_key` or `schedule_key` already delivered; retries should stop without a second Slack send. |
| `no_eligible_lead` | Neither fresh `new` leads nor eligible `notified` backlog leads are available; n8n should send an ops note. |
| `locked` | Another run owns the `delivery_locks` row; retry later. |

### 2. Inspect the Database
Use `psql` (Postgres) or `sqlite3` (SQLite) to see what's happening under the hood.

#### Postgres (Recommended)
```bash
# Total count of collected projects vs. those that were enriched
docker exec groupscout_postgres psql -U groupscout -d groupscout -c "SELECT status, count(*) FROM leads GROUP BY status;"

# Look at the most recent "skipped" leads and why they were skipped
docker exec groupscout_postgres psql -U groupscout -d groupscout -c "SELECT title, priority_score, priority_reason FROM leads WHERE status = 'skipped' ORDER BY created_at DESC LIMIT 10;"
```

#### SQLite
```bash
# Total count of collected projects vs. those that were enriched
sqlite3 groupscout.db "SELECT status, count(*) FROM leads GROUP BY status;"
```

### 3. Adjust Thresholds
If the filters are too strict, you can relax them in your `.env` file or environment variables:

- `MIN_PERMIT_VALUE_CAD`: Lower this (e.g., to `100000`) to include smaller construction projects.
- `ENRICHMENT_THRESHOLD`: Set this to `0` to enrich **every** project that passes the collector filter, regardless of keywords or location.
- `PRIORITY_ALERT_THRESHOLD`: Set this to a lower value (e.g., `5`) if you want more "instant alerts" in Slack for moderate-priority leads.

### 4. Verify External Tools
- **Richmond/Delta Permits**: These require `pdftotext` (from `poppler-utils`) to be installed on your system or inside your Docker container. If it's missing, these collectors will log an error and return 0 leads.
- **n8n / API Triggers**: If you are triggering the pipeline via `/run`, ensure your `API_TOKEN` is correct.
- **n8n direct leads**: If you are posting pre-enriched leads to `/n8n/webhook`, send `PriorityScore` on the `0-10` scale. Legacy `0-100` values are normalized by the backend before storage and Slack/email delivery.

---

## 🖥 Backend Plus UI Docker Troubleshooting

Use [Backend And Frontend Docker E2E](../planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md) for the full current smoke runbook.

### Testing Stack Sanity Checks

For UI/API smoke and Postgres-backed testing, these checks are enough before deeper debugging:

```bash
curl -i --max-time 5 http://localhost:8080/health
curl -i --max-time 5 http://localhost:3001/healthz
curl -i --max-time 5 http://localhost:3001/
```

Backend `/health` should return `200` with `"database":"ok"`, and the UI product dev server should return `200` for `/healthz` and `/`. This is enough for UI/API smoke and Postgres-backed testing.

If only the Ollama field is unhealthy, do not block UI/API smoke. Do block LLM enrichment and quality testing until the API can reach Ollama and required models are listed:

```bash
docker compose ps ollama
docker compose logs ollama --tail=50
docker exec groupscout_ollama ollama list
```

### Admin Setup Token Is Rejected

Default admin auth is setup-token based. Check the current token source:

```bash
curl -i http://localhost:8080/api/auth/status
docker exec groupscout_app sh -lc 'cat data/admin-setup-token'
```

Common causes:

- The backend container was rebuilt or restarted and active sessions were lost.
- A file-backed setup token already succeeded once and rotated; read `data/admin-setup-token` again.
- `ADMIN_SETUP_TOKEN` is set in the backend environment, so the file value is ignored and the env value is the active setup token.
- The browser is still using stale UI assets; check that `http://localhost:3001/` references `/assets/app.js?v=admin-login-1`.

Successful login sets `groupscout_session`. Logout must call `POST /api/auth/logout`; browser JavaScript should not try to clear the HttpOnly cookie directly.

### `compose.dev.yml` Fails By Itself

The UI repo's `compose.dev.yml` is an override for the backend Compose file. It expects backend service `groupscout` and network `groupscout_net`.

Run it merged with the backend file:

```bash
cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui

docker compose -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  config --quiet
```

### `localhost:3001` Does Not Serve The Production UI

That is expected. Port `3001` is the UI development/product dev server for the Compose override. Use it for `/healthz` and generated app-shell checks, not for proving the production static/proxy server.

Use the D4 production UI container on port `3002` for static assets and `/api/*` proxy smoke checks.

### Port `3000` Is Busy

The backend Compose stack publishes Grafana on host port `3000`. Use `3001` for the D3 health harness and `3002` for the D4 production UI smoke container.

### D4 Proxy Returns `502`

The UI production container cannot reach the backend target.

Check:

- The container was started with `--network groupscout_groupscout_net`.
- `UI_API_PROXY_TARGET=http://groupscout:8080`.
- Backend service `groupscout` is running: `docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml ps groupscout`.

### D4 Proxy Returns `404`

The proxy reached the backend, but the route does not exist or the path is wrong. On backend `main`, `/api/system` currently returns `404` through the D4 proxy; once protected UI API routes are present, expect `401` without a browser session. A `502` means the UI proxy could not reach the backend target.

### UI Container Has Backend Secrets

Do not pass backend `.env` files into UI containers. The browser-visible UI must not receive `API_TOKEN`, provider keys, Slack tokens, email-provider keys, database URLs, Ollama endpoints, or `UI_SESSION_SECRET`.

---

## ❓ FAQ

### Why is there only 1 lead?
If `/run` was called with `guarantee_one_lead:true` and `delivery_mode:"exactly_one"`, one lead is intentional: the Sunday/Wednesday cadence sends exactly one eligible lead and records it in `lead_deliveries`.

This usually happens if:
- You've already run the pipeline today, and all other permits are now **duplicates**.
- The latest weekly report only has 1 permit that meets both the **commercial sub-type** AND the **$500k+ value** criteria.
- Most other permits have a **score of 0** (meaning they aren't in Richmond/YVR and didn't contain any high-value keywords like "pipeline", "film", or "concrete").

### Why did Slack show `98/10`?
That means an external pre-enriched payload posted to `/n8n/webhook` supplied a raw `PriorityScore` such as `98` while Slack renders priority as `/10`. The backend now normalizes webhook scores before storage and notification: `90` becomes `9`, `98` becomes `10`, values above `100` become `10`, and negatives become `0`.

For new n8n workflows, send `PriorityScore` as `0-10`. Use `/ingest` instead of `/n8n/webhook` when GroupScout should score and enrich a raw source item.

### How do I re-process old leads?
If you want to force the pipeline to re-process everything (e.g., after changing the scoring logic), you must clear the database:
```bash
# This stops containers, removes volumes, and deletes local SQLite db
make clear
```
*(Or manually delete rows from `raw_projects` and `leads` tables).*
