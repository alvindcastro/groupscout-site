# GroupScout — Setup & Run Guide

End-to-end walkthrough: from prerequisites to n8n scheduling the pipeline automatically.

---

## Prerequisites

| Tool | Required for | Install |
|---|---|---|
| Go 1.26+ | Local server mode | https://go.dev/dl/ |
| `pdftotext` | PDF permit scraping | Bundled with Git for Windows at `C:\Program Files\Git\mingw64\bin\pdftotext.exe` — no extra install needed |
| Docker + Docker Compose | Docker mode | https://docs.docker.com/get-docker/ |

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
ADMIN_AUTH_ENABLED=true
ADMIN_SETUP_TOKEN_FILE=data/admin-setup-token

# Privacy (Optional)
PII_STRIP=true             # Set to true to redact emails and phone numbers from raw audit trail
```

Generate a secure `API_TOKEN`:
```bash
openssl rand -hex 32
```

> Everything else has sensible defaults. See `.env.example` for the full list.

Admin auth defaults on. If `ADMIN_SETUP_TOKEN` is not set, the backend creates `ADMIN_SETUP_TOKEN_FILE` and uses that value for `/admin/login`. File-backed setup tokens rotate after successful login. See [ADMIN_AUTH.md](./ADMIN_AUTH.md) for the complete login, logout, and token-rotation flow.

---

## Postgres Setup (recommended)
1. `docker compose up postgres -d` and wait for healthy: `docker compose ps`
2. Set DATABASE_URL=postgres://groupscout:groupscout@localhost:5432/groupscout in .env
3. `go run ./cmd/server/ --run-once` — migrations run automatically on first boot

## SQLite (local dev only)
DATABASE_URL=groupscout.db — no Docker required.

---

## Data Retention & Privacy

GroupScout keeps a `raw_inputs` audit trail to ensure traceability of leads.

### 1. Retention Policy
To prevent infinite database growth, raw inputs that are NOT referenced by any active leads can be purged.
- **Auto-purge**: (Optional) Configure a cron job to call the purge method via the internal API (if exposed) or manually via SQL.
- **Safety**: The purge logic uses a `NOT EXISTS` check on the `leads` table to ensure that any raw data linked to an actual lead is NEVER deleted.

### 2. PII Stripping
Enable `PII_STRIP=true` in your `.env` to redact sensitive information (emails, phone numbers) from raw payloads before they are stored in the database. This is recommended for compliance with local data privacy laws.

---

## Ollama Setup (local LLM — no API cost)

Run enrichment with a local model instead of Claude or Gemini. No data leaves your server. Requires roughly 2-5 GB RAM for the model. Use [OLLAMA_SETUP.md](./OLLAMA_SETUP.md) for current Docker service names, model pull/list/push commands, environment variables, and troubleshooting.

### Compare providers

Use the smoketest tool to compare Ollama vs Claude on the same fixtures:

```bash
COMPARE_LLM_PROVIDER=claude go run ./cmd/smoketest/ -compare-providers
```

This prints a side-by-side table of `priority_score`, `estimated_crew_size`, and `out_of_town_crew_likely` from both providers.

---

## Step 2 — Choose a run mode

### Option A — Local Go server

```bash
# Install dependencies
go mod download

# Start the HTTP server (stays running, listens on :8080)
go run cmd/server/main.go
```

Expected output:
```
INFO  GroupScout server started  addr=:8080
INFO  Database ready
```

**Or run the pipeline once and exit (no server):**
```bash
go run cmd/server/main.go --run-once
```

---

### Option B — Docker Compose (recommended)

Starts GroupScout + **Postgres (with pgvector/pgvector:pg17)** + n8n + Prometheus + Grafana + Loki in one command.

```bash
docker compose up -d
```

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

Run these checks after `docker compose up -d`:

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
docker compose up -d --build
```

> If `go mod download` fails during build, check that the Go version in `Dockerfile` matches `go.mod`. `go.mod` currently declares `go 1.26` — the Dockerfile must use `golang:1.26-alpine` or higher.

---

#### Docker — Teardown

Stop containers (keep volumes/images):
```bash
docker compose down
```

Stop and remove everything including volumes:
```bash
docker compose down --rmi all --volumes
```

---

## Step 3 — Verify the server is up

```bash
curl -i --max-time 5 http://localhost:8080/health
```

Expected response: HTTP `200` with JSON similar to:

```json
{"status":"ok","database":"ok","ollama":"ok"}
```

The `ollama` value may also be `degraded` or `unavailable`; that blocks LLM enrichment verification, not basic health or UI/API smoke. A non-`ok` database value blocks the backend.

---

## Step 4 — Test the pipeline manually

Trigger a full collect → enrich → Slack run:

```bash
curl -X POST http://localhost:8080/run \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

Expected response: HTTP `200` with `Pipeline completed successfully`. That confirms the run finished; it does not prove that new leads were found.

Check the run result in logs:

```bash
docker compose logs groupscout --tail=100
```

Interpret the result:

- `sent leads to Slack count=N`: leads were delivered to Slack.
- `no new leads to notify`: the run succeeded, but filters or deduplication left no fresh leads.
- `Unauthorized`: the `API_TOKEN` in n8n or curl does not match the backend.
- `slack notify`: Slack delivery failed; check `SLACK_WEBHOOK_URL`.
- collector or `pdftotext` errors: rebuild the image and inspect collector settings.

> **Tip:** Lower `MIN_PERMIT_VALUE_CAD=100000` in `.env` during testing to get more results through the filter.

---

## Step 5 — Set up n8n

### 5a — Open n8n

- **Docker mode**: http://localhost:5678
- **Local Go + separate n8n**: install n8n via `npm install -g n8n` and run `n8n start`

Complete the initial account setup on first launch.

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

### 5c — Create the workflow

1. Go to **Workflows** → **New Workflow**
2. Add a **Schedule** trigger node:
   - **Trigger Interval**: `Weeks`
   - **Days of the Week**: `Sunday`, `Wednesday`
   - **Time**: `09:00`
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

Click **Test Workflow** (or the play button on the Schedule node) to fire it immediately without waiting for Sunday or Wednesday.

Check:
- The HTTP Request node shows a `200 OK` response
- A new Slack message appears with leads

### 5e — Make the twice-weekly lead send reliable

The workflow above reliably runs GroupScout, but it does not by itself guarantee one lead every Sunday and Wednesday. GroupScout may return zero new leads after deduplication, low-value filtering, or previous notification.

For the current n8n-only MVP:

1. Add a preflight HTTP Request node for `GET http://groupscout:8080/health`.
2. Keep the scheduled `POST /run` node as the source-of-truth pipeline trigger.
3. For now, use a log check or manual database check to distinguish `sent leads to Slack` from `no new leads to notify`; the current `/run` response is plain text and does not include lead counts.
4. Add an error/no-leads branch that sends an operational Slack message when no qualified lead is available.
5. Store a cadence key such as `lead-cadence:YYYY-MM-DD:sunday` in n8n's data store so retries do not duplicate the same cadence notification.

For a true guaranteed one-lead cadence, plan backend work for a delivery log, fallback lead selector, run lock, and machine-readable `/run` result. The backend should own "which one lead is eligible" and "has this cadence already delivered"; n8n should own the Sunday/Wednesday schedule and failure routing.

Completed workflow asset: `groupscout-site-yfl` added the importable Sunday/Wednesday n8n workflow JSON, Docker import smoke notes, health preflight, cadence idempotency key, and no-leads/failure Slack branch. Tracked follow-up: `groupscout-site-fuc` owns the backend delivery guarantee: delivery log, fallback selector, machine-readable run/delivery result, and run lock.

Workflow asset: [`../workflows/n8n/sunday-wednesday-lead-cadence.json`](../workflows/n8n/sunday-wednesday-lead-cadence.json). Import and Docker smoke notes live in [`../workflows/n8n/README.md`](../workflows/n8n/README.md).

## Step 6 — Migrate Data (SQLite to Postgres)

If you've been using SQLite (`groupscout.db`) and want to move your data to a new PostgreSQL instance:

1.  **Ensure Postgres is running** and accessible (e.g., via `docker compose up -d`).
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
| `/digest` | POST | Send weekly email digest (`?to=email@example.com`) |
| `/n8n/webhook` | POST | Push a single lead from an external n8n workflow |

All endpoints except `/health` require `Authorization: Bearer YOUR_API_TOKEN`.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `401 Unauthorized` | Check `API_TOKEN` in `.env` matches the Bearer value in n8n |
| `Connection refused` (Docker n8n → groupscout) | Use `http://groupscout:8080` not `localhost` — they share a Docker network |
| `Connection refused` (local n8n → local Go) | Use `http://host.docker.internal:8080` on Mac/Windows |
| No leads generated | Lower `MIN_PERMIT_VALUE_CAD` to `100000` for testing |
| PDF parse errors | Confirm `pdftotext` is on your PATH: `pdftotext -v` |
| `pdftotext not found` (local) | Add `C:\Program Files\Git\mingw64\bin` to your system PATH |
| `pdftotext not found` (Docker) | `poppler-utils` is now included in the Docker image. If you see this, run `docker compose up -d --build`. |
| `go mod download` fails during Docker build | Go version mismatch — `Dockerfile` must use `golang:1.26-alpine` to match `go.mod` |
| `Failed to initialize: protocol not available` | Docker Desktop WSL integration not enabled for your distro — see Docker Desktop → Settings → Resources → WSL Integration |
| `permission denied` on Docker socket | Run `sudo usermod -aG docker $USER && newgrp docker` |
| Docker Desktop UI stuck on stale compose entry | Run `docker compose down` from the project directory, then restart Docker Desktop |
