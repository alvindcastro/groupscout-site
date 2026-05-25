# Coolify Deployment Guide

This guide covers the Hetzner + Coolify path from Phase 26. Use it when uptime matters more than the free home-server path in [HOME_DEPLOY.md](./HOME_DEPLOY.md).

## Target Shape

Recommended baseline:

- Hetzner CX32 or similar VPS running Ubuntu 24.04.
- Coolify installed on the VPS.
- GroupScout deployed from the backend source repo's `docker-compose.yml`.
- Persistent services: `groupscout`, `alertd`, `postgres`, `n8n`, Prometheus/Grafana/Loki as needed.
- Public HTTPS for the API and operator surfaces through Coolify's proxy and Let's Encrypt.
- Scheduled Postgres backups plus a documented restore smoke.

## 1. Provision The VPS

1. Create a Hetzner Cloud project.
2. Provision a CX32 in the closest practical region.
3. Select Ubuntu 24.04.
4. Add an SSH key.
5. Point a DNS `A` record to the server IP, for example:
   - `server.example.com`
   - `alertd.example.com`
   - `n8n.example.com`
   - `grafana.example.com`

Initial hardening:

```sh
sudo apt-get update
sudo apt-get -y upgrade
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Keep the Coolify dashboard itself behind a strong password and, where possible, an allowlist or VPN.

## 2. Install Coolify

Run the Coolify installer from the VPS shell:

```sh
curl -fsSL https://cdn.coolify.io/install.sh | bash
```

After installation:

1. Open the Coolify dashboard URL printed by the installer.
2. Create the first admin user.
3. Add the VPS as the local server if it is not already registered.
4. Confirm Docker is healthy from the Coolify server view.

## 3. Add The GroupScout Application

In Coolify:

1. Create a new **Application**.
2. Select the GitHub backend repository.
3. Use the branch that should deploy, normally `main`.
4. Select Docker Compose deployment.
5. Use the repository's `docker-compose.yml`.
6. Set automatic deploys on push only after the first manual deploy passes health checks.

If the backend repo is private, connect the GitHub app/token from Coolify's source setup screen.

## 4. Service Mapping

Map services from Compose as persistent services. Names may vary with the backend source repo, but the deployment should preserve these roles:

| Service | Public? | Purpose | Health check |
|---|---:|---|---|
| `groupscout` | Yes | Main automation API, `/run`, `/digest`, `/ingest`, `/n8n/webhook`, raw audit routes; `/api/*` UI routes remain planned under `groupscout-site-eqm` | `GET /health` returns HTTP 200 with `"database":"ok"` |
| `alertd` | Optional | YVR disruption daemon and Slack slash-command endpoint | `GET /health` on the alertd internal port |
| `postgres` | No | Postgres with pgvector | `SELECT 1; SELECT extname FROM pg_extension WHERE extname='vector';` |
| `n8n` | Optional | Workflow scheduler and integrations | n8n UI loads and can reach `http://groupscout:8080/health` |
| `prometheus` | Internal/guarded | Metrics scraping | Targets are up |
| `grafana` | Guarded | Dashboards | Login reachable only behind auth/allowlist |
| `loki` / `promtail` | No | Log aggregation | Grafana can query logs |

Do not expose `postgres`, `loki`, or internal Prometheus scrape endpoints publicly.

## 5. Environment Variables

Set these in Coolify's environment variable editor. Use strong generated values for secrets.

Required:

```env
DATABASE_URL=postgres://groupscout:<strong-password>@postgres:5432/groupscout?sslmode=disable
POSTGRES_DB=groupscout
POSTGRES_USER=groupscout
POSTGRES_PASSWORD=<strong-password>
API_TOKEN=<openssl-rand-hex-32>
SLACK_WEBHOOK_URL=<slack-incoming-webhook>
```

Common production settings:

```env
PORT=8080
JSON_LOG=true
PII_STRIP=true
ENRICHMENT_ENABLED=true
ENRICHMENT_THRESHOLD=1
PRIORITY_ALERT_THRESHOLD=9
MIN_PERMIT_VALUE_CAD=500000
```

Email digest:

```env
RESEND_API_KEY=<resend-api-key>
```

Admin/operator session:

```env
ADMIN_AUTH_ENABLED=true
ADMIN_SETUP_TOKEN=<temporary-setup-token>
ADMIN_SESSION_TTL_HOURS=24
```

Alertd:

```env
ALERTD_PORT=8081
SLACK_BOT_TOKEN=<xoxb-token-if-alertd-actions-are-enabled>
```

Ollama inside Compose:

```env
OLLAMA_ENABLED=true
OLLAMA_ENDPOINT=http://ollama:11434
OLLAMA_MODEL=mistral
OLLAMA_EXTRACTION_ENABLED=true
OLLAMA_SCORING_ENABLED=true
OLLAMA_ALERT_COPY_ENABLED=true
```

n8n workflow imports:

```env
GROUPSCOUT_API_BASE_URL=http://groupscout:8080
GROUPSCOUT_API_TOKEN=<same-value-as-API_TOKEN>
GROUPSCOUT_OPS_SLACK_WEBHOOK_URL=<ops-slack-webhook>
```

Never place backend secrets into browser-visible frontend config or static assets.

## 6. Domains And SSL

For the API:

1. In Coolify, attach `server.example.com` to the `groupscout` service.
2. Set the service port to `8080`.
3. Enable HTTPS/Let's Encrypt.
4. Verify:

```sh
curl -fsS https://server.example.com/health
curl -Iv http://server.example.com/health
```

The first command should return JSON with `"database":"ok"`. The second should redirect to HTTPS.

For `alertd`, expose it only if Slack slash commands or external alert hooks need to reach it. Otherwise keep it internal and Slack-webhook-only.

For `n8n` and Grafana, require strong passwords and prefer an allowlist, VPN, or Cloudflare Access in front of the domain.

## 7. pgvector Health

Run from a shell inside the Postgres container or a Coolify database console:

```sh
psql "$DATABASE_URL" -c "SELECT 1;"
psql "$DATABASE_URL" -c "SELECT extname FROM pg_extension WHERE extname = 'vector';"
```

Expected:

- `SELECT 1` returns one row.
- The extension query returns `vector`.

If `vector` is missing, verify the Compose image is `pgvector/pgvector:pg17` or another Postgres image with pgvector installed, then re-run migrations.

## 8. Backups

Use at least one of these backup layers.

### Coolify Scheduled Backup

1. Open the Postgres resource in Coolify.
2. Configure a scheduled backup.
3. Store backups outside the VPS, for example Hetzner Object Storage or another S3-compatible bucket.
4. Use weekly as the minimum cadence; daily is better once production usage starts.
5. Turn on backup failure notifications.

### Manual pg_dump Backup

From the VPS:

```sh
mkdir -p ~/groupscout-backups
docker exec groupscout_postgres pg_dump -U groupscout groupscout > ~/groupscout-backups/groupscout_$(date +%Y%m%d).sql
```

Restore smoke into a disposable database before trusting the backup:

```sh
docker exec groupscout_postgres createdb -U groupscout groupscout_restore_smoke
cat ~/groupscout-backups/groupscout_20260525.sql | docker exec -i groupscout_postgres psql -U groupscout groupscout_restore_smoke
docker exec groupscout_postgres psql -U groupscout groupscout_restore_smoke -c "SELECT count(*) FROM leads;"
docker exec groupscout_postgres dropdb -U groupscout groupscout_restore_smoke
```

Keep at least one restore-smoke result in the operations log before calling deployment complete.

## 9. First Deploy Verification

Run these checks after the first Coolify deploy:

```sh
curl -fsS https://server.example.com/health
curl -i -X POST https://server.example.com/run \
  -H "Authorization: Bearer $API_TOKEN"
```

For cadence delivery, import `backend/docs/workflows/n8n/sunday-wednesday-lead-cadence.json` into n8n, confirm it is inactive after import, set `GROUPSCOUT_API_BASE_URL`, `GROUPSCOUT_API_TOKEN`, and `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL`, then run the health preflight and guaranteed `/run` test. Expected delivery statuses are `sent`, `duplicate`, `no_eligible_lead`, or `locked`.

Then verify the containers:

```sh
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker logs --tail=100 groupscout_app
docker logs --tail=100 groupscout_alertd
```

Acceptance:

- `groupscout` is a persistent container and restarts automatically.
- `alertd` is a persistent container and does not run on a scale-to-zero surface.
- `/health` is HTTP 200 with database OK.
- Postgres has pgvector.
- At least one scheduled backup exists.
- One restore smoke has been run.
- Public observability surfaces are protected.

## 10. Troubleshooting

**Coolify deploy cannot find Compose services**

- Confirm the Compose file path is the backend repo root `docker-compose.yml`.
- Confirm the selected branch contains the current Compose file.
- Run `docker compose config --quiet` in the backend source repo before pushing.

**API returns database unavailable**

- Check `DATABASE_URL` uses the Compose service name `postgres`, not `localhost`.
- Confirm `POSTGRES_PASSWORD` matches the Postgres service env.
- Run the pgvector health checks above.

**HTTPS does not issue**

- Confirm DNS points to the VPS public IP.
- Confirm ports 80 and 443 are open in Hetzner firewall and `ufw`.
- Check the Coolify proxy logs for ACME/Let's Encrypt errors.

**n8n cannot call GroupScout**

- From the n8n container, call `http://groupscout:8080/health`.
- Confirm `GROUPSCOUT_API_BASE_URL=http://groupscout:8080`.
- Confirm `GROUPSCOUT_API_TOKEN` matches backend `API_TOKEN`.

**Grafana or n8n is exposed too broadly**

- Add an access layer before sharing the URL.
- Prefer an allowlist, VPN, or Cloudflare Access for internal operations tools.
