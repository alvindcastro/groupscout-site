# Docker Guide — GroupScout

This guide provides technical details for running and troubleshooting GroupScout using Docker and Docker Compose. Docker Compose remains the known-good baseline. If you are testing Podman compatibility, start with [PODMAN_MIGRATION.md](./PODMAN_MIGRATION.md) and keep any unverified runtime differences there.

> **Status:** The Compose layout, profiles, pinned images and `/health` version described here are IaC Phase 1 (Beads `groupscout-site-6ah.2`), merged to backend `main` and verified on Podman 2026-10-14. The plan for the rest of the infrastructure is in [INFRASTRUCTURE_AS_CODE.md](../planning/INFRASTRUCTURE_AS_CODE.md).

## 🚀 Running with Docker Compose

GroupScout's Compose setup starts the lead generation server, disruption monitor, database and n8n, plus optional Ollama and monitoring services.

### Compose Layout

| File | Purpose | Loaded when |
|---|---|---|
| `docker-compose.yml` | Base: every service, the local image build (stamped with the git SHA), pinned images, healthchecks, log rotation. **No host ports.** | Always |
| `docker-compose.override.yml` | Dev host ports (`8080`, `8081`, `5432`, `5678`, `9090`, `3000`, `3100`) | Automatically, when no `-f` flag is given |
| `deploy/compose/compose.prod.yml` | Production: registry image by tag (no local build), required secrets, ports bound to `127.0.0.1`, memory limits, `JSON_LOG=true`, Ollama off by default | Only with `-f docker-compose.yml -f deploy/compose/compose.prod.yml` |

Commands that pass `-f` explicitly skip the dev override. For example, the UI repo's `-f docker-compose.yml -f compose.dev.yml` flows still build and run the backend, but don't publish its host ports. Add `-f docker-compose.override.yml` after the base file if you also need `localhost:8080`.

`deploy/config/` holds config mounted read-only into the containers, so a fresh volume comes back fully configured:

- `n8n/` holds the workflow bootstrap and the tracked workflows; see [the workflow runbook](../workflows/n8n/README.md).
- `prometheus/prometheus.yml` is the scrape config.
- `grafana/` holds the provisioned datasources and dashboards.
- `alloy/config.alloy` configures container log shipping.

### Profiles

| Profile | Services | Started by |
|---|---|---|
| none (core) | `postgres`, `groupscout`, `alertd`, `n8n` | Always |
| `llm` | `ollama`, `ollama-init` | `make docker-up` (default), or `COMPOSE_PROFILES` containing `llm` |
| `obs` | `prometheus`, `grafana`, `loki`, `alloy` | `make docker-up` (default), or `COMPOSE_PROFILES` containing `obs` |

`make docker-up` exports `COMPOSE_PROFILES=llm,obs`. Use `make docker-up COMPOSE_PROFILES=` for core services only. Plain `docker compose up -d` and `podman compose up -d` read `COMPOSE_PROFILES` from `.env`. Without it, they start the core services only, and `/health` reports Ollama as `degraded`. Naming a profiled service directly (`docker compose up -d ollama`) starts it regardless.

## Current Operational Runbooks

Use [BACKEND_FOR_UI_TESTING.md](../planning/ui/BACKEND_FOR_UI_TESTING.md) for local backend startup notes, [Backend And Frontend Docker E2E](../planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md) for backend plus external UI smoke, [TESTING.md](./TESTING.md) for test commands, and [OLLAMA_SETUP.md](./OLLAMA_SETUP.md) for Ollama model operations.

### 1. Prerequisites
- **Docker Desktop** (for Windows/macOS) or **Docker Engine + Docker Compose** (for Linux). Compose 2.20 or newer is required for optional `depends_on` (`required: false`) and the production overlay's `!reset` tag.
- **WSL2** (strongly recommended for Windows users).

### 2. Environment Setup
Ensure you have a `.env` file in the project root. With the age key, render it with `make secrets-render` from the encrypted `deploy/secrets/prod.sops.env` ([SECRETS.md](./SECRETS.md)); otherwise copy `.env.example`. The minimum values for the Compose stack:
```env
CLAUDE_API_KEY=sk-ant-...
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/XXX/YYY/ZZZ
SLACK_BOT_TOKEN=xoxb-...
API_TOKEN=your_secure_token_here
COMPOSE_PROFILES=llm,obs
AUDIT_RETENTION_ENABLED=false
AUDIT_RETENTION_DAYS=30
AUDIT_RETENTION_INTERVAL_HOURS=24
AUDIT_RETENTION_RUN_ON_START=true
```

Compose builds the container's `DATABASE_URL` from `POSTGRES_PASSWORD`. A `DATABASE_URL` in `.env` only affects `go run` outside Docker. These optional values fall back to dev defaults when blank, and the production overlay requires all three:

| Variable | Dev default | Notes |
|---|---|---|
| `POSTGRES_PASSWORD` | `groupscout` | Only applied when the `pgdata` volume is first created. Use a URL-safe value (`openssl rand -hex 24`), since it is embedded in `DATABASE_URL`. To rotate on an existing volume, also run `ALTER USER groupscout PASSWORD '...'`. |
| `N8N_ENCRYPTION_KEY` | key stored in `n8n_data` | When set, it must match `/home/node/.n8n/config` inside the volume, or n8n refuses to start. Read it with `docker compose exec n8n cat /home/node/.n8n/config`. |
| `GRAFANA_ADMIN_PASSWORD` | `admin` | Only applied when the `grafana_data` volume is first created. Changing it later needs `grafana cli admin reset-admin-password` ([SECRETS.md](./SECRETS.md#rotating-a-secret)). |

`alertd` receives only the variables it reads (`SLACK_BOT_TOKEN`, `ALERTD_PORT`, and its `OLLAMA_*` settings), not the whole `.env`.

### 3. Start the Stack
```bash
make docker-up
```
This builds the backend image stamped with the current git SHA (`--build`), then starts the core services plus the `llm` and `obs` profiles in the background. The plain Compose equivalent:

```bash
GROUPSCOUT_VERSION=$(git rev-parse --short HEAD) COMPOSE_PROFILES=llm,obs docker compose up -d --build
```

Validate every profile combination and the production overlay without starting anything:

```bash
make compose-check
```

Podman uses the same targets:

```bash
make docker-up COMPOSE="podman compose"
make compose-check COMPOSE="podman compose"
```

The Compose file sets `name: groupscout`, so `-p groupscout` is no longer required (it is still harmless). Do not assume every Docker command below is Podman-compatible. Traefik socket discovery, Ollama GPU access, and host aliases have separate caveats in [PODMAN_MIGRATION.md](./PODMAN_MIGRATION.md).

### 4. Service Overview

| Service | Dev host port | Profile | Image | Healthcheck |
|---|---|---|---|---|
| `groupscout` | 8080 | core | local build (`groupscout:local`) | `GET /health` |
| `alertd` | 8081 | core | local build (same image) | `GET /health` |
| `postgres` | 5432 | core | `pgvector/pgvector:0.8.7-pg17-bookworm` | `pg_isready` |
| `n8n` | 5678 | core | `n8nio/n8n:2.43.5` | `GET /healthz` |
| `ollama` | not published | `llm` | `ollama/ollama:0.40.4` | `ollama list` |
| `prometheus` | 9090 | `obs` | `prom/prometheus:v3.15.0` | none |
| `grafana` | 3000 | `obs` | `grafana/grafana:13.2.4` | none |
| `loki` | 3100 | `obs` | `grafana/loki:3.7.8` | none |
| `alloy` | not published | `obs` | `grafana/alloy:v1.20.1` | none |

Check health with `docker compose ps`. Healthy core services show `(healthy)` in the status column. `GET /health` returns a `version` field with the git SHA the running image was built from. `"version":"dev"` means the image was built without `GROUPSCOUT_VERSION`:

```bash
curl -s http://localhost:8080/health
# {"database":"ok","ollama":"ok","status":"ok","version":"bdcff09"}
```

Images are pinned. n8n and Grafana migrate their databases forward on upgrade and cannot be downgraded, so back up `n8n_data` (and `grafana_data` if it holds anything you care about) before bumping their tags. pgvector pins the `-bookworm` variant on purpose. If the `pg17` tag moves to a newer Debian base, the glibc collation change can corrupt text indexes on an existing data directory.

The backend containers run as an unprivileged user (uid `10001`).

### n8n Local Access

Open n8n at `http://localhost:5678` after the stack starts.

```bash
docker compose up -d n8n
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep groupscout_n8n
docker logs --tail 40 groupscout_n8n
```

Expected logs include `Editor is now accessible via: http://localhost:5678`. If the local sign-in password is lost, use the reset and recovery steps in [Troubleshooting](./TROUBLESHOOTING.md#6-recover-local-n8n-sign-in).

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

Container logs rotate at 10 MB, keeping 5 files per container.

### Rebuild and Restart
`make docker-up` always rebuilds, so the running image matches your checkout. To rebuild just the backend services:
```bash
GROUPSCOUT_VERSION=$(git rev-parse --short HEAD) docker compose up -d --build groupscout alertd
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
`docker compose down` only acts on services in the active profiles. `make docker-down` exports `COMPOSE_PROFILES=llm,obs`, so it stops everything.
```bash
# Stop and remove containers (all profiles)
make docker-down

# Stop and remove containers, volumes, and images
COMPOSE_PROFILES=llm,obs docker compose down --rmi all --volumes

# Clear everything (database, volumes, and local SQLite)
make clear
```

### Reset and Run Pipeline
To completely reset the environment and run one pipeline pass to verify the flow:
```bash
make start-fresh
```

## 🏭 Production Overlay

`deploy/compose/compose.prod.yml` is the starting point for a real deployment, which IaC Phases 5–7 will automate:

```bash
export GROUPSCOUT_IMAGE=ghcr.io/alvindcastro/groupscout GROUPSCOUT_IMAGE_TAG=<git sha>
docker compose -f docker-compose.yml -f deploy/compose/compose.prod.yml up -d
```

It refuses to render unless these are set: `GROUPSCOUT_IMAGE`, `GROUPSCOUT_IMAGE_TAG`, `POSTGRES_PASSWORD`, `API_TOKEN`, `N8N_ENCRYPTION_KEY`, `GRAFANA_ADMIN_PASSWORD`. It never builds from source; the base file's `build` is removed with `!reset`. Only `groupscout` (8080), `n8n` (5678) and `grafana` (3000) are published, all on `127.0.0.1` for SSH-tunnel access, because Docker-published ports bypass UFW. Registry images arrive with IaC Phase 4 (`groupscout-site-6ah.5`).

## 🖥 Frontend UI Docker Smoke

The operator UI lives in a separate repo at `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`. For the current backend-plus-frontend Docker runbook, see [Backend And Frontend Docker E2E](../planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md).

Key points:

- Backend Compose service `groupscout` joins `groupscout_net`. It publishes `http://localhost:8080` through the dev override, which `make docker-up` and plain `docker compose up` load automatically.
- Grafana uses host port `3000`, so UI examples use `3001` for the development health harness and `3002` for the production static/proxy smoke container.
- The UI repo's `compose.dev.yml` is an override, not a standalone Compose file.
- The UI Compose service is the Phase 13 product dev server. It serves generated `web/dist` assets and `/healthz`; use the UI D4 production image when you need the production static/proxy runtime. The production image starts from a clean pinned Node base and contains only `package.json` and `web/`.
- First backend boot can be slow when the `llm` profile is active, because `groupscout` waits for `ollama` and `ollama-init`. Without that profile, the backend starts as soon as Postgres is healthy.

Keep the full command sequence in the E2E runbook so D3/D4 smoke expectations do not drift across docs.

UI Docker gate status, reconciled 2026-06-13 (`groupscout-site-crz`): `make smoke-ui-docker-e2e` and `internal/smoke` are now present. Run the gate from the backend repo with `make smoke-ui-docker-e2e` once both Compose stacks are up. Open follow-up: `groupscout-site-yyj` tracks Prometheus metrics, collector freshness, and dashboard documentation.

## 🧠 Ollama Model Management

Models are stored in a persistent volume (`groupscout_ollama_data`). Ollama runs only with the `llm` profile. Use [OLLAMA_SETUP.md](./OLLAMA_SETUP.md) for model pull/list/push commands, default-model configuration, and Ollama troubleshooting.

## 💾 Backups

It is recommended to back up your database and Ollama models periodically. IaC Phase 8 (`groupscout-site-6ah.9`) replaces these manual steps with scheduled offsite backups and a restore smoke.

### Automated Backup Script
Run the provided script to back up all named volumes:
```bash
./scripts/backup-volumes.sh
```
This creates compressed tarballs in the `./backups` directory. The script is Docker-only. On Podman, use `podman volume export <volume> -o <file>.tar`.

### Manual Database Dump (Postgres)
```bash
docker compose exec postgres pg_dumpall -c -U groupscout > dump.sql
```

## 📊 Monitoring & Logging

GroupScout includes an optional observability stack (the `obs` profile).

### Logging (Loki + Alloy)
Grafana Alloy reads every container's logs through the Docker API socket and ships them to `loki`, labelled `container`, `compose_service` and `compose_project`. On the Podman machine, `/var/run/docker.sock` links to the rootless Podman socket, so the same config works there. Promtail, which needed `/var/lib/docker/containers`, was removed in IaC Phase 2.
- **First start**: Alloy backfills each container's full log history. Loki rejects entries older than one week, so a burst of `timestamp too old` errors in `docker compose logs alloy` right after the first start is expected. Read positions persist in the `alloy_data` volume.
- **View logs in Grafana**: Open `http://localhost:3000`, go to **Explore**, and select the **Loki** data source.
- **Filter by container**: Use `{container="groupscout_app"}` or `{container="groupscout_ollama"}`.

### Metrics (Prometheus + Grafana)
Metrics are collected by `prometheus` and can be visualized in `grafana`.
- **Prometheus UI**: `http://localhost:9090`
- **Grafana UI**: `http://localhost:3000` (login `admin` with `GRAFANA_ADMIN_PASSWORD`, default `admin` on a fresh volume)

Prometheus scrapes `groupscout:8080/metrics` every 30 s (`deploy/config/prometheus/prometheus.yml`). On every start, Grafana provisions the Prometheus and Loki datasources and the **GroupScout Overview** dashboard (folder *GroupScout*) from `deploy/config/grafana/`. The dashboard shows service status, uptime, memory, CPU, goroutines, Ollama calls and latency, and app and n8n logs. Edit dashboards as JSON in git; UI edits to provisioned dashboards are not saved. The app currently exports Go runtime metrics plus `ollama_*` (only when Ollama is used). Pipeline counters for collected, enriched and notified leads are tracked in `groupscout-site-yyj`.

#### Custom Ollama Panel
To monitor Ollama resource usage, add a panel with these queries:
- **CPU Usage**: `rate(container_cpu_usage_seconds_total{container="groupscout_ollama"}[5m])`
- **Memory Usage**: `container_memory_usage_bytes{container="groupscout_ollama"}`

## 🔍 Troubleshooting

### 1. `pdftotext not found` (Alpine Container)
GroupScout requires the `pdftotext` utility for scraping certain PDF sources. The Alpine-based Dockerfile includes `poppler-utils` in the final stage to provide this functionality.
- **Symptom:** `richmond_permits` and `delta_permits` collectors failing.
- **Fix:** Rebuild with `make docker-up`, then confirm `/health` reports the SHA you expect.

### 2. Go Version Mismatch
- **Symptom:** `go mod download` fails during `docker build`.
- **Fix:** The builder image is pinned to a full patch release (`golang:1.26.9-alpine3.24`). It must satisfy the `go` directive in `go.mod`. Bump the Dockerfile pin when `go.mod` moves forward.

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
- **Symptom:** `groupscout` container fails to start because it can't reach Postgres, or logs `password authentication failed`.
- **Fix:** The `groupscout` service waits for `postgres` to be healthy. Check `docker compose logs postgres` to ensure the database initialized correctly. If you set `POSTGRES_PASSWORD` after the `pgdata` volume already existed, the database still has the old password. Either unset it, or run `ALTER USER groupscout PASSWORD '<new>'` inside `postgres`.

### 7. Missing or Low Lead Count
- **Symptom:** Pipeline runs successfully but returns fewer leads than expected.
- **Fix:** See the [Troubleshooting Guide](./TROUBLESHOOTING.md) for details on adjusting thresholds (`MIN_PERMIT_VALUE_CAD`, `ENRICHMENT_THRESHOLD`) and checking the database for skipped or duplicate records.

### 8. Ollama or Grafana Not Running After `docker compose up`
- **Symptom:** `docker compose ps` lists only `postgres`, `groupscout`, `alertd` and `n8n`, or `/health` shows `"ollama":"degraded"`.
- **Fix:** Ollama and the monitoring services are behind the `llm` and `obs` profiles. Use `make docker-up`, add `COMPOSE_PROFILES=llm,obs` to `.env`, or run `COMPOSE_PROFILES=llm,obs docker compose up -d`.

### 9. n8n Refuses to Start: `Mismatching encryption keys`
- **Symptom:** `groupscout_n8n` restarts in a loop and logs `Mismatching encryption keys`.
- **Fix:** `N8N_ENCRYPTION_KEY` in `.env` differs from the key stored in `n8n_data`. Blank the variable to use the stored key, or copy the stored value from `/home/node/.n8n/config` into `.env`.

### 10. n8n Restarts In A Loop After A Workflow Change
- **Symptom:** `groupscout_n8n` keeps restarting, and `docker compose logs n8n` shows `[groupscout-n8n-bootstrap]` lines followed by an n8n error.
- **Fix:** The workflow bootstrap stops the container when an import or publish fails, so a broken workflow file can't leave the schedule quietly dead. Fix the JSON under `deploy/config/n8n/workflows/`. `go test ./internal/smoke/` checks the id, name and `active` fields. Then run `docker compose up -d n8n`.
