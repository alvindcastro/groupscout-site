# GroupScout — Setup & Run Guide

End-to-end walkthrough: from prerequisites to n8n scheduling the pipeline automatically.

---

## Prerequisites

| Tool | Required for | Install |
|---|---|---|
| Go 1.26+ | Local server mode | https://go.dev/dl/ |
| `pdftotext` | PDF permit scraping | Bundled with Git for Windows at `C:\Program Files\Git\mingw64\bin\pdftotext.exe` — no extra install needed |
| Docker + Docker Compose | Production and CI command baseline | https://docs.docker.com/get-docker/ |
| Podman CLI + Docker Compose provider | Validated local container mode | See [PODMAN_MIGRATION.md](./PODMAN_MIGRATION.md). On Windows, install `RedHat.Podman` and standalone `Docker.DockerCompose`, then run `podman machine init` and `podman machine start`. |

---

## Step 1 — Create your `.env` file

Copy the example and fill in your keys:

```bash
cp .env.example .env
```

Minimum required values:

```env
CLAUDE_API_KEY=sk-ant-...
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/XXX/YYY/ZZZ
API_TOKEN=your_secure_token_here
DATABASE_URL=groupscout.db # Or postgres://user:pass@localhost:5432/db
# Planned admin UI auth; not present in the inspected backend source snapshot yet.
# ADMIN_AUTH_ENABLED=true
# ADMIN_SETUP_TOKEN_FILE=data/admin-setup-token

# Privacy (Optional)
PII_STRIP=true             # Set to true to redact emails and phone numbers from raw audit trail
```

Generate a secure `API_TOKEN`:
```bash
openssl rand -hex 32
```

For the Docker Compose stack (Option B), `.env` also controls which optional services start and the container secrets. Since IaC Phase 1, blank values fall back to dev defaults that match existing volumes:

```env
COMPOSE_PROFILES=llm,obs   # Ollama + Prometheus/Grafana/Loki; blank = core services only
POSTGRES_PASSWORD=         # dev default: groupscout (applies only when pgdata is first created)
N8N_ENCRYPTION_KEY=        # blank keeps the key stored in n8n_data; if set, it must match it
GRAFANA_ADMIN_PASSWORD=    # dev default: admin (applies only when grafana_data is first created)
```

See [DOCKER.md](./DOCKER.md#2-environment-setup) for what each one does on an existing volume.

> Everything else has sensible defaults. See `.env.example` for the full list.

Contract-only admin auth defaults on after the `/api/auth/*` implementation lands. The inspected backend source snapshot does not currently create `ADMIN_SETUP_TOKEN_FILE` or expose `/admin/login`/`/api/auth/*`; see [ADMIN_AUTH.md](./ADMIN_AUTH.md) for the target login, logout, and token-rotation flow.

---

## Postgres Setup (recommended)
1. `docker compose up postgres -d` and wait for healthy: `docker compose ps`
2. Set DATABASE_URL=postgres://groupscout:groupscout@localhost:5432/groupscout in .env
3. `go run ./cmd/server/ --run-once` — migrations run automatically on first boot

## SQLite (local dev only)
DATABASE_URL=groupscout.db — no Docker required.

---

## Podman Local Runtime

Local Podman smoke passed on 2026-07-01. Docker Compose remains the production and CI command baseline, but the backend stack can be validated locally with Podman after the runtime is initialized:

```bash
podman --version
podman compose version
podman info
cd /mnt/c/Users/alvin/GolandProjects/groupscout
COMPOSE="podman compose" make docker-up
curl -i --max-time 5 http://localhost:8080/health
podman exec groupscout_postgres psql -U groupscout -d groupscout -c "SELECT 1;"
```

On Windows PowerShell, refresh PATH after installing `Docker.DockerCompose` so `podman compose` can find `docker-compose.exe`. Git Bash can run the backend-owned smoke script when the Winget Docker Compose package directory is appended to PATH; plain WSL cannot execute the Windows `podman.exe` in this local setup. Use [PODMAN_MIGRATION.md](./PODMAN_MIGRATION.md) for the exact UI E2E smoke command and port notes.

---

## Data Retention & Privacy

GroupScout keeps a `raw_inputs` audit trail to ensure traceability of leads.

### 1. Retention Policy
To prevent infinite database growth, raw inputs that are NOT referenced by any lead can be purged.
- **Auto-purge**: Set `AUDIT_RETENTION_ENABLED=true` in server mode. The worker defaults to `AUDIT_RETENTION_DAYS=30`, `AUDIT_RETENTION_INTERVAL_HOURS=24`, and `AUDIT_RETENTION_RUN_ON_START=true`.
- **Manual purge**: Run `go run ./cmd/server audit-retention purge --days 30` from the backend source repo. The command prints JSON with `cutoff`, `deleted`, `started_at`, and `ended_at`.
- **Safety**: The purge logic uses a `NOT EXISTS` check on the `leads` table to ensure that any raw data linked to a lead is NEVER deleted.

### 2. PII Stripping
Enable `PII_STRIP=true` in your `.env` to redact sensitive information (emails, phone numbers) from raw payloads before they are stored in the database. This is recommended for compliance with local data privacy laws.

---

## Ollama Setup (local LLM — no API cost)

Run enrichment with a local model instead of Claude or Gemini. No data leaves your server. Requires roughly 2-5 GB RAM for the model. Use [OLLAMA_SETUP.md](./OLLAMA_SETUP.md) for current Docker service names, model pull/list/push commands, environment variables, and troubleshooting.

### Compare providers

Use the smoketest tool to compare Ollama vs Claude on the same fixtures:

```bash
COMPARE_LLM_PROVIDER=claude go run ./cmd/tools/smoketest/ -compare-providers
```

This prints a side-by-side table of `priority_score`, `estimated_crew_size`, and `out_of_town_crew_likely` from both providers.

---

## Step 2 — Choose a run mode

### Option A — Local Go server

```bash
# Install dependencies
go mod download

# Start the HTTP server (stays running, listens on :8080)
go run ./cmd/server
```

Expected output:
```
INFO  GroupScout server started  addr=:8080
INFO  Database ready
```

**Or run the pipeline once and exit (no server):**
```bash
go run ./cmd/server --run-once
```

Run the package (`./cmd/server`), not `cmd/server/main.go`. The server is split across several files, and naming one file compiles only that file.

---

### Option B — Docker Compose (recommended)

Starts GroupScout + **Postgres (pgvector `0.8.7-pg17-bookworm`)** + n8n, plus Ollama (`llm` profile) and Prometheus + Grafana + Loki + Alloy (`obs` profile), in one command. The Compose layout, profiles and pinned image versions are described in [DOCKER.md](./DOCKER.md#compose-layout). Testing Podman instead of Docker? Use [PODMAN_MIGRATION.md](./PODMAN_MIGRATION.md) first. The core app/Postgres/n8n path is the initial target; Promtail Docker socket mounts, Traefik Docker provider examples, host aliases, and Ollama GPU support need separate validation.

```bash
make docker-up
```

`make docker-up` builds the backend image stamped with the current git SHA and starts the `llm` and `obs` profiles. Plain `docker compose up -d --build` starts only the core services unless `COMPOSE_PROFILES` is set in `.env`.

> **WSL2 users:** Modern Docker installations use the `docker compose` (v2 plugin) command. If you get a "'docker-compose' could not be found" error, ensure you are using the space-separated command `docker compose` instead of the hyphenated `docker-compose`. Ensure **Docker Desktop → Settings → Resources → WSL Integration** is enabled for your distro.

> **Permission denied on Docker socket?** Run `sudo usermod -aG docker $USER && newgrp docker` then retry.

Services that come up:

| Service | URL | Purpose |
|---|---|---|
| GroupScout API | http://localhost:8080 | Pipeline trigger + health check |
| Postgres | localhost:5432 | Primary database (when configured) |
| n8n | http://localhost:5678 | Workflow scheduler |
| Grafana | http://localhost:3000 | Dashboards + log viewer |
| Prometheus | http://localhost:9090 | Metrics |
| Loki | http://localhost:3100 | Log aggregation |

Prometheus, Grafana and Loki come from the `obs` profile. Ollama (`llm` profile) is internal only and has no host port.

---

#### Docker — Log Commands

Check container status:
```bash
docker compose ps
```

Follow GroupScout logs in real time:
```bash
docker compose logs -f groupscout
```

View recent logs (last 50 lines):
```bash
docker compose logs groupscout --tail=50
```

Follow logs for all services:
```bash
docker compose logs -f
```

View logs for a specific service:
```bash
docker compose logs n8n --tail=30
docker compose logs grafana --tail=30
```

#### Docker — Operator Quick Check

Run these checks after `make docker-up`:

```bash
docker compose ps
curl -i --max-time 5 http://localhost:8080/health
docker exec groupscout_postgres psql -U groupscout -d groupscout -c "SELECT 1;"
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
```

Expected result:

- `docker compose ps` shows `groupscout`, `postgres`, and `n8n` running.
- `/health` returns HTTP `200` and JSON with `"database":"ok"`.
- `"ollama":"ok"` is required for LLM enrichment claims. `"ollama":"degraded"` or `"ollama":"unavailable"` is not a blocker for basic backend/UI/API smoke.
- Postgres returns a single `1`.
- The n8n container can reach `http://groupscout:8080/health`.

If any check fails, inspect:

```bash
docker compose logs --tail=100 groupscout postgres n8n ollama
```

---

#### Docker — Reading a Pipeline Run

After triggering `/run`, check `docker compose logs groupscout --tail=50`. A healthy run looks like:

```
INFO  pipeline triggered via HTTP /run
INFO  active collectors  count=8  names="[richmond_permits delta_permits ...]"
INFO  starting collection...  collector=richmond_permits
INFO  processing latest report  source=richmond  count=64
INFO  starting collection...  collector=creativebc
INFO  collection complete  collector=creativebc  count=5
...
INFO  new lead inserted  title="..."  score=9
INFO  enrichment complete  new_leads=3
INFO  sent leads to Slack  count=1
```

**Known warnings (safe to ignore):**
- `"RESEND_API_KEY" variable is not set` — expected if email digest isn't configured
- `"RICHMOND_PERMITS_URL" variable is not set` — uses the hardcoded default URL

**Known errors (fixed):**
- `pdftotext not found` on richmond_permits and delta_permits — `poppler-utils` is now included in the `Dockerfile`.

---

#### Docker — Rebuild After Code Changes

```bash
make docker-up
```

`make docker-up` always rebuilds, so `/health` reports the SHA of your checkout.

> If `go mod download` fails during build, check that the Go version in `Dockerfile` satisfies `go.mod`. `go.mod` currently declares `go 1.26`, and the Dockerfile pins `golang:1.26.9-alpine3.24`.

---

#### Docker — Teardown

Stop containers in every profile (keep volumes/images):
```bash
make docker-down
```

Stop and remove everything including volumes:
```bash
COMPOSE_PROFILES=llm,obs docker compose down --rmi all --volumes
```

Plain `docker compose down` only stops services in the active profiles.

---

## Step 3 — Verify the server is up

```bash
curl -i --max-time 5 http://localhost:8080/health
```

Expected response: HTTP `200` with JSON similar to:

```json
{"database":"ok","ollama":"ok","status":"ok","version":"bdcff09"}
```

The `ollama` value may also be `degraded` or `unavailable`. That blocks LLM enrichment verification, not basic health or UI/API smoke; `degraded` usually means the `llm` profile isn't running. A non-`ok` database value blocks the backend. `version` is the git SHA the image was built from (`dev` for `go run` or builds without `GROUPSCOUT_VERSION`). Compare it with `git rev-parse --short HEAD` to confirm the container runs your checkout.

---

## Step 4 — Test the pipeline manually

Trigger a full collect → enrich → Slack run:

```bash
curl -X POST http://localhost:8080/run \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

Expected response: HTTP `200` with JSON similar to:

```json
{"status":"success","new_leads":1,"notified_leads":1}
```

Check the run result in logs:

```bash
docker compose logs groupscout --tail=100
```

Interpret the result:

- `sent leads to Slack count=N`: leads were delivered to Slack.
- `no new leads to notify`: the run succeeded, but filters, deduplication, or an earlier cadence delivery left no fresh `new` leads. This is expected if today's only lead is already `notified`.
- `Unauthorized`: the `API_TOKEN` in n8n or curl does not match the backend.
- `slack notify`: Slack delivery failed; check `SLACK_WEBHOOK_URL`.
- collector or `pdftotext` errors: rebuild the image and inspect collector settings.

> **Tip:** Lower `MIN_PERMIT_VALUE_CAD=100000` in `.env` during testing to get more results through the filter.

---

## Step 5 — Set up n8n

### 5a — Open n8n

- **Docker mode**: http://localhost:5678
- **Local Go + separate n8n**: install n8n via `npm install -g n8n` and run `n8n start`

Complete the initial account setup on first launch. For the Docker stack, run these checks before opening the browser:

```bash
cd /mnt/c/Users/alvin/GolandProjects/groupscout
docker compose up -d n8n
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep groupscout_n8n
docker logs --tail 40 groupscout_n8n
```

Expected: the container is `Up`, port `5678` is published, and logs include `Editor is now accessible via: http://localhost:5678`.

If the local sign-in password is lost, stop the running n8n container before resetting user state so SQLite is not open. Use the same n8n version that `docker-compose.yml` pins. A newer image would migrate the volume's database, and n8n can't migrate it back:

```bash
docker stop groupscout_n8n
docker run --rm --volumes-from groupscout_n8n n8nio/n8n:2.43.5 user-management:reset
docker start groupscout_n8n
```

If the UI still shows only **Sign In** after that reset, the owner row may still exist without a usable local login. Use the recovery procedure in [Troubleshooting](./TROUBLESHOOTING.md#6-recover-local-n8n-sign-in).

---

### 5b — Create a credential

n8n needs your `API_TOKEN` to call GroupScout.

1. Go to **Credentials** → **Add Credential**
2. Search for **Header Auth**
3. Fill in:
   - **Name**: `GroupScout API`
   - **Header Name**: `Authorization`
   - **Value**: `Bearer YOUR_API_TOKEN`
4. Click **Save**

---

### 5c — Import the Sunday/Tuesday/Thursday 6 PM workflow

With the provided Docker Compose stack there is nothing to import. Since IaC Phase 2, n8n's bootstrap imports and publishes the tracked workflow (backend repo `deploy/config/n8n/workflows/sunday-wednesday-lead-cadence.json`) every time n8n starts with the workflow missing or the file changed. For an n8n outside the Compose stack, follow **Importing Outside The Compose Stack** in [`../workflows/n8n/README.md`](../workflows/n8n/README.md), set the `GROUPSCOUT_API_BASE_URL`, `GROUPSCOUT_API_TOKEN`, and `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL` environment variables or node values, and run **Test Workflow**.

In the provided Docker Compose stack, those n8n environment values are injected from backend `.env` when the `n8n` container is created. `GROUPSCOUT_API_TOKEN` maps to `API_TOKEN`, `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL` maps to `SLACK_WEBHOOK_URL`, and `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` allows workflow expressions to read `$env.*`. If any of these values change, run `docker compose up -d n8n`, then verify they are present without printing secrets:

```bash
docker exec groupscout_n8n sh -lc 'for k in N8N_BLOCK_ENV_ACCESS_IN_NODE GROUPSCOUT_API_BASE_URL GROUPSCOUT_API_TOKEN GROUPSCOUT_OPS_SLACK_WEBHOOK_URL; do v=$(printenv "$k"); if [ -n "$v" ]; then echo "$k=SET"; else echo "$k=MISSING"; fi; done'
docker exec groupscout_n8n node -e "fetch('http://groupscout:8080/health').then(r=>r.text()).then(console.log)"
docker exec groupscout_n8n n8n list:workflow --active=true
```

After restart, export the workflow and confirm it is active with `triggerAtDay:[0,2,4]`, `triggerAtHour:18`, and `timezone:"America/Vancouver"`. Also confirm the `/run` HTTP node uses the guaranteed cadence body; if the export still contains `JSON.stringify({})`, the live workflow is stale. Force a re-import with `docker exec groupscout_n8n rm /home/node/.n8n/groupscout-workflows.sha256 && docker compose restart n8n`.

When the cadence timing changes, edit the tracked workflow JSON first, update `triggerAtDay`, `triggerAtHour`, and `triggerAtMinute`, then re-import, publish, restart, and re-export to verify the live schedule.

In the n8n UI, open **GroupScout Lead Cadence (Sun/Tue/Thu 6 PM)** and verify the visible graph is:

```txt
Schedule Sun Tue Thu 6 PM
  -> Build Cadence Key
  -> Duplicate Cadence?
  -> Health Preflight
  -> Classify Health
  -> Health OK?
  -> Trigger GroupScout Run
  -> Classify Run Result
  -> Delivered?
  -> Mark Delivered
```

The failure branches should lead to **Ops Slack Health Failure** or **Ops Slack No Lead Or Run Failure**. The top-right workflow toggle must be **Active** before the schedule can fire.

Manual fallback:

1. Go to **Workflows** → **New Workflow**.
2. Add a **Schedule** trigger node:
   - **Trigger Interval**: `Weeks`
   - **Days of the Week**: `Sunday`, `Tuesday`, `Thursday`
   - **Time**: `18:00`
   - **Timezone**: `America/Vancouver` unless the operator team works from another fixed timezone
3. Add an **HTTP Request** node and connect it to the Schedule node:
   - **Method**: `POST`
   - **URL**:
     - Compose n8n to Compose backend: `http://groupscout:8080/run`
     - Host n8n to host backend: `http://localhost:8080/run`
     - Containerized n8n to host backend: `http://host.docker.internal:8080/run`
   - **Authentication**: `Predefined Credential Type` → `Header Auth` → select `GroupScout API`
4. **Save** the workflow
5. Toggle **Active** (top-right switch) to enable it

---

### 5d — Test the workflow manually

Click **Test Workflow** (or the play button on the Schedule node) to fire it immediately without waiting for the next scheduled slot.

For the live container, prefer the UI trigger over `n8n execute`; the CLI execute path can fail while the running n8n instance already owns its task-broker port.

Check:
- The HTTP Request node shows a `200 OK` response
- The backend JSON includes `delivery_status`, `notified_leads`, and `message`; `delivery_status:"sent"` or `"duplicate"` means the cadence is handled, and `delivery_status:"no_eligible_lead"` should flow to the ops Slack no-lead branch

### 5e — Make the scheduled lead send reliable

The tracked importable workflow already includes the reliable Sunday/Tuesday/Thursday path: health preflight, guaranteed cadence `/run` body, workflow-level cadence duplicate guard, and ops Slack branching. The manual fallback above is a basic scheduler with the same backend run mode.

For a manual Sunday/Tuesday/Thursday cadence:

1. Add a preflight HTTP Request node for `GET http://groupscout:8080/health`.
2. Keep the scheduled `POST /run` node as the source-of-truth pipeline trigger.
3. Send the same cadence JSON body used by the tracked workflow so `/run` receives `guarantee_one_lead:true`, `delivery_mode:"all_eligible"`, and the cadence key fields.
4. Add an error/no-leads branch that sends an operational Slack message when no qualified lead is available.
5. Store the cadence key in n8n's data store so workflow-level retries can stop early after the backend reports `delivery_status:"sent"` or `"duplicate"`.

The backend owns collection, enrichment, Slack fan-out, and machine-readable `/run` results. n8n owns the Sunday/Tuesday/Thursday schedule, workflow-level duplicate guard, and failure routing.

Completed workflow asset: `groupscout-site-yfl` added the importable Sunday/Wednesday n8n workflow JSON, Docker import smoke notes, health preflight, cadence key, and no-leads/failure Slack branch. `groupscout-site-ar1` restored the workflow to normal multi-lead `/run` behavior after the exactly-one cadence proved too narrow for scheduled sends.

Workflow asset: backend repo `deploy/config/n8n/workflows/sunday-wednesday-lead-cadence.json`, imported automatically by the n8n bootstrap. The runbook is [`../workflows/n8n/README.md`](../workflows/n8n/README.md).

## Step 6 — Migrate Data (SQLite to Postgres)

If you've been using SQLite (`groupscout.db`) and want to move your data to a new PostgreSQL instance:

1.  **Ensure Postgres is running** and accessible (e.g., via `docker compose up -d postgres`).
2.  **Run the migration script**:

    ```bash
    go run cmd/tools/migrate_db/main.go \
      --sqlite groupscout.db \
      --postgres "postgres://groupscout:groupscout@localhost:5432/groupscout"
    ```

    *   Use `--dry-run` to preview the row counts without writing to Postgres.
    *   The script is idempotent — it uses `ON CONFLICT DO NOTHING` to prevent duplicates if run multiple times.
    *   It handles all tables in the correct foreign key order, including `lead_embeddings` and `outreach_log`.

---

## Endpoints Reference

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Health check — no auth required |
| `/run` | POST | Run full pipeline (collect → enrich → notify) |
| `/digest` | POST | Send weekly email digest (`?to=alvin.dcastro@gmail.com` until Resend domain verification is complete) |
| `/ingest` | POST | Push one raw project/event payload through `EnrichOne()` |
| `/n8n/webhook` | POST | Push a pre-enriched single lead from an external n8n workflow |

All endpoints except `/health` require `Authorization: Bearer YOUR_API_TOKEN`.

For `/n8n/webhook`, send `PriorityScore` on the Slack/internal `0-10` scale. Legacy percent-style `0-100` scores are accepted for compatibility and normalized before storage and Slack/email display, so a stale workflow that sends `90` renders as `9/10` instead of `90/10`. Prefer `/ingest` when the payload is raw source data that GroupScout should score itself.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `401 Unauthorized` | Check `API_TOKEN` in `.env` matches the Bearer value in n8n |
| `Connection refused` (Docker n8n → groupscout) | Use `http://groupscout:8080` not `localhost` — they share a Docker network |
| `Connection refused` (local n8n → local Go) | Use `http://host.docker.internal:8080` on Mac/Windows |
| Slack shows an impossible score such as `98/10` | Update to the score-normalizing backend and send `/n8n/webhook` `PriorityScore` as `0-10`; old `0-100` payloads are normalized after the fix. |
| No leads generated | Lower `MIN_PERMIT_VALUE_CAD` to `100000` for testing |
| PDF parse errors | Confirm `pdftotext` is on your PATH: `pdftotext -v` |
| `pdftotext not found` (local) | Add `C:\Program Files\Git\mingw64\bin` to your system PATH |
| `pdftotext not found` (Docker) | `poppler-utils` is now included in the Docker image. If you see this, rebuild with `make docker-up`. |
| `go mod download` fails during Docker build | Go version mismatch — the pinned builder (`golang:1.26.9-alpine3.24`) must satisfy `go.mod`; bump the Dockerfile pin when `go.mod` moves forward |
| `/health` shows `"ollama":"degraded"` and no `groupscout_ollama` container | Ollama is in the `llm` profile — use `make docker-up` or set `COMPOSE_PROFILES=llm,obs` in `.env` |
| n8n restarts with `Mismatching encryption keys` | `N8N_ENCRYPTION_KEY` in `.env` differs from the key in `n8n_data` — blank it or copy the stored key from `/home/node/.n8n/config` |
| `go run cmd/server/main.go` fails with `undefined: ...` | Run the package instead: `go run ./cmd/server` |
| `Failed to initialize: protocol not available` | Docker Desktop WSL integration not enabled for your distro — see Docker Desktop → Settings → Resources → WSL Integration |
| `podman compose` cannot find a provider | Install standalone `Docker.DockerCompose`, refresh PATH, and confirm `podman compose version` prints the Compose version. |
| Podman host port returns an empty response | Retry with an explicit IPv4 host mapping such as `-p 127.0.0.1:3004:3000`; the 2026-07-01 smoke hit a rootless relay issue on `3002:3000`. |
| `permission denied` on Docker socket | Run `sudo usermod -aG docker $USER && newgrp docker` |
| Docker Desktop UI stuck on stale compose entry | Run `docker compose down` from the project directory, then restart Docker Desktop |
