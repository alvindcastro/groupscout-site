# Docker Guide — GroupScout

This guide provides technical details for running and troubleshooting GroupScout using Docker and Docker Compose.

## 🚀 Running with Docker Compose

GroupScout includes a `docker-compose.yml` that starts the full stack, including the lead generation server, disruption monitor, database, and monitoring tools.

## Current Operational Runbooks

Use [BACKEND_FOR_UI_TESTING.md](../planning/ui/BACKEND_FOR_UI_TESTING.md) for local backend startup notes, [Backend And Frontend Docker E2E](../planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md) for backend plus external UI smoke, [TESTING.md](./TESTING.md) for test commands, and [OLLAMA_SETUP.md](./OLLAMA_SETUP.md) for Ollama model operations.

### 1. Prerequisites
- **Docker Desktop** (for Windows/macOS) or **Docker Engine + Docker Compose** (for Linux).
- **WSL2** (strongly recommended for Windows users).

### 2. Environment Setup
Ensure you have a `.env` file in the project root with the minimum required variables:
```env
CLAUDE_API_KEY=sk-ant-...
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/XXX/YYY/ZZZ
SLACK_BOT_TOKEN=xoxb-...
API_TOKEN=your_secure_token_here
DATABASE_URL=postgres://groupscout:groupscout@postgres:5432/groupscout
AUDIT_RETENTION_ENABLED=false
AUDIT_RETENTION_DAYS=30
AUDIT_RETENTION_INTERVAL_HOURS=24
AUDIT_RETENTION_RUN_ON_START=true
```

### 3. Start the Stack
```bash
docker compose up -d
```
This starts all services in the background.

### 4. Service Overview

| Service | Port | Description |
|---|---|---|
| `groupscout` | 8080 | Main lead generation server (cmd/server) |
| `alertd` | 8081 | Airport disruption monitor (cmd/alertd) |
| `postgres` | 5432 | Database with `pgvector` extension |
| `n8n` | 5678 | Workflow automation / scheduler |
| `prometheus`| 9090 | Metrics collection |
| `grafana` | 3000 | Dashboards and log viewer |
| `loki` | 3100 | Log aggregation |
| `promtail`| 9080 | Log scraper (internal) |

## 🛠 Common Commands

### Check Container Status
```bash
docker compose ps
```

### View Logs
```bash
# Follow GroupScout API logs
docker compose logs -f groupscout

# Follow alertd logs
docker compose logs -f alertd

# View last 50 lines for all services
docker compose logs --tail=50
```

### Rebuild and Restart
If you change the Go code, you must rebuild the containers:
```bash
docker compose up -d --build
```

### Raw Audit Retention
The `groupscout` service passes the raw audit retention environment through to the backend container. Keep `AUDIT_RETENTION_ENABLED=false` unless you want the server to purge unreferenced `raw_inputs` automatically.

```bash
AUDIT_RETENTION_ENABLED=true docker compose up -d --build groupscout
docker compose logs groupscout --tail=50
```

For a one-time purge from the running container:

```bash
docker compose exec groupscout ./groupscout audit-retention purge --days 30
```

The purge preserves every `raw_inputs` row referenced by `leads.raw_input_id`.

### Teardown
```bash
# Stop and remove containers
docker compose down

# Stop and remove containers, volumes, and images
docker compose down --rmi all --volumes

# Clear everything (database, volumes, and local SQLite)
make clear
```

### Reset and Run Pipeline
To completely reset the environment and run one pipeline pass to verify the flow:
```bash
make start-fresh
```

## 🖥 Frontend UI Docker Smoke

The operator UI lives in a separate repo at `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`. For the current backend-plus-frontend Docker runbook, see [Backend And Frontend Docker E2E](../planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md).

Key points:

- Backend Compose service `groupscout` publishes `http://localhost:8080` and joins `groupscout_net`.
- Grafana uses host port `3000`, so UI examples use `3001` for the development health harness and `3002` for the production static/proxy smoke container.
- The UI repo's `compose.dev.yml` is an override, not a standalone Compose file.
- The UI Compose service is the Phase 13 product dev server. It serves generated `web/dist` assets and `/healthz`; use the UI D4 production image when you need the production static/proxy runtime.
- First backend boot can be slow because `groupscout` depends on `ollama` and `ollama-init`.

Keep the full command sequence in the E2E runbook so D3/D4 smoke expectations do not drift across docs.

UI Docker gate status, 2026-05-25: the UI repo has a `smoke-ui-e2e` production profile, but the inspected backend source snapshot does not currently expose `make smoke-ui-docker-e2e` or `internal/smoke`. Restore or merge the backend-owned Phase 38 smoke target in `groupscout-site-crz` before treating the E2E gate as runnable. Open follow-up: `groupscout-site-yyj` tracks Prometheus metrics, collector freshness, and dashboard documentation for collected/enriched/notified/failure counts.

## 🧠 Ollama Model Management

Models are stored in a persistent volume (`groupscout_ollama_data`). Use [OLLAMA_SETUP.md](./OLLAMA_SETUP.md) for model pull/list/push commands, default-model configuration, and Ollama troubleshooting.

## 💾 Backups

It is recommended to back up your database and Ollama models periodically.

### Automated Backup Script
Run the provided script to back up all named volumes:
```bash
./scripts/backup-volumes.sh
```
This creates compressed tarballs in the `./backups` directory.

### Manual Database Dump (Postgres)
```bash
docker exec -t groupscout_postgres pg_dumpall -c -U groupscout > dump.sql
```

## 📊 Monitoring & Logging

GroupScout includes a full observability stack.

### Logging (Loki + Promtail)
Logs from all containers are automatically scraped by `promtail` and sent to `loki`.
- **View logs in Grafana**: Open `http://localhost:3000`, go to **Explore**, and select the **Loki** data source.
- **Filter by container**: Use `{container="groupscout_app"}` or `{container="groupscout_ollama"}`.

### Metrics (Prometheus + Grafana)
Metrics are collected by `prometheus` and can be visualized in `grafana`.
- **Prometheus UI**: `http://localhost:9090`
- **Grafana UI**: `http://localhost:3000` (default login: `admin/admin`)

#### Custom Ollama Panel
To monitor Ollama resource usage, add a panel with these queries:
- **CPU Usage**: `rate(container_cpu_usage_seconds_total{container="groupscout_ollama"}[5m])`
- **Memory Usage**: `container_memory_usage_bytes{container="groupscout_ollama"}`

## 🔍 Troubleshooting

### 1. `pdftotext not found` (Alpine Container)
GroupScout requires the `pdftotext` utility for scraping certain PDF sources. The Alpine-based Dockerfile now includes `poppler-utils` in the final stage to provide this functionality.
- **Symptom:** `richmond_permits` and `delta_permits` collectors failing.
- **Fix:** Ensure you are using the latest `Dockerfile` and have rebuilt your container with `docker compose up -d --build`.

### 2. Go Version Mismatch
- **Symptom:** `go mod download` fails during `docker build`.
- **Fix:** Ensure the `FROM golang:X.XX-alpine` version in `Dockerfile` matches the `go 1.XX` version in `go.mod`.

### 3. Container Communication
- **Symptom:** n8n cannot connect to the `groupscout` server.
- **Fix:** When n8n is running inside Docker, use `http://groupscout:8080` instead of `localhost:8080`. Containers on the same Docker network use their service names as hostnames.

### 4. WSL2 Specifics (Windows)
- **Symptom:** `docker-compose: command not found`.
- **Fix:** Modern Docker uses `docker compose` (v2 plugin). Ensure **Docker Desktop → Settings → Resources → WSL Integration** is enabled for your specific Linux distro.

### 5. Permission Denied on Docker Socket (Linux)
- **Symptom:** `docker compose` requires `sudo`.
- **Fix:** Add your user to the `docker` group:
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```

### 6. Database Connection Issues
- **Symptom:** `groupscout` container fails to start because it can't reach Postgres.
- **Fix:** The `groupscout` service is configured to depend on `postgres` being healthy. Check `docker compose logs postgres` to ensure the database initialized correctly.

### 7. Missing or Low Lead Count
- **Symptom:** Pipeline runs successfully but returns fewer leads than expected.
- **Fix:** See the [Troubleshooting Guide](./TROUBLESHOOTING.md) for details on adjusting thresholds (`MIN_PERMIT_VALUE_CAD`, `ENRICHMENT_THRESHOLD`) and checking the database for skipped or duplicate records.
