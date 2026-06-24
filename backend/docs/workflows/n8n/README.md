# GroupScout n8n Workflows

## Daily (Except Saturday) Lead Cadence

Import `sunday-wednesday-lead-cadence.json` into n8n for the daily internal lead send. The filename and workflow id (`groupscout-sunday-wednesday-lead-cadence`) are retained for stability; the schedule now runs daily except Saturday.

The workflow:

- runs every day except Saturday at 09:00 in `America/Vancouver` (`triggerAtDay: [0,1,2,3,4,5]`);
- writes a cadence key such as `lead-cadence:2026-05-27:wednesday` in workflow static data and stops duplicate runs only after a cadence is marked delivered;
- calls `GET /health` before running the pipeline;
- calls `POST /run` with a bearer token and a cadence-delivery body containing `guarantee_one_lead`, `delivery_mode`, `cadence_key`, `schedule_key`, and `idempotency_key`;
- sends an operational Slack message when health fails, the run fails, or the backend returns no notified leads;
- retries transient HTTP node failures up to three times;
- stops silently on duplicate cadence-key runs after delivery.

Required n8n environment variables:

| Variable | Purpose |
|---|---|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | Must be `false` so workflow expressions can read `$env.*` values. |
| `GROUPSCOUT_API_BASE_URL` | GroupScout base URL. Use `http://groupscout:8080` from Compose n8n to Compose backend. |
| `GROUPSCOUT_API_TOKEN` | Bearer token matching backend `API_TOKEN`. |
| `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL` | Slack incoming webhook for operational failures/no-leads notices. |

The backend Docker Compose service injects these values into the `n8n` container from `.env`: `GROUPSCOUT_API_TOKEN` comes from `API_TOKEN`, and `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL` comes from `SLACK_WEBHOOK_URL`. After editing `.env` or `docker-compose.yml`, recreate n8n with `docker compose up -d n8n` so the live container receives the new environment.

The JSON keeps secrets out of the repository. If these environment variables are not set, imported nodes show placeholder values that must be replaced before activation.

The tracked export includes a stable workflow `id` because the Docker-backed n8n smoke stack currently runs n8n `2.14.2`, whose CLI import requires a non-empty workflow id.

This workflow calls `/run` for cadence delivery. For separate event-driven source pushes, use `POST /ingest` when GroupScout should enrich one raw project/event payload through `EnrichOne()`. Use `/n8n/webhook` only for pre-enriched lead-shaped payloads that should be inserted directly.

### Docker Import Smoke

From the coordination repo root:

```sh
docker cp backend/docs/workflows/n8n/sunday-wednesday-lead-cadence.json groupscout_n8n:/tmp/sunday-wednesday-lead-cadence.json
docker exec groupscout_n8n n8n import:workflow --input=/tmp/sunday-wednesday-lead-cadence.json
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
docker exec groupscout_n8n sh -lc 'for k in N8N_BLOCK_ENV_ACCESS_IN_NODE GROUPSCOUT_API_BASE_URL GROUPSCOUT_API_TOKEN GROUPSCOUT_OPS_SLACK_WEBHOOK_URL; do v=$(printenv "$k"); if [ -n "$v" ]; then echo "$k=SET"; else echo "$k=MISSING"; fi; done'
```

Expected health output includes `"database":"ok"`, and every env check should print `SET` without exposing secret values.

Then open n8n at `http://localhost:5678`, confirm the workflow is inactive after import, set any missing env-backed values or credentials, run **Test Workflow**, and activate it only after the health and `/run` nodes return the expected status. CLI activation is:

```sh
docker exec groupscout_n8n n8n update:workflow --id=groupscout-sunday-wednesday-lead-cadence --active=true
docker restart groupscout_n8n
docker exec groupscout_n8n n8n export:workflow --id=groupscout-sunday-wednesday-lead-cadence --output=/tmp/groupscout-cadence-review.json
docker exec groupscout_n8n sh -lc "grep -o '\"active\":[^,}]*\|\"triggerAtDay\":\[[^]]*\]\|\"triggerAtHour\":[0-9]*\|\"triggerAtMinute\":[0-9]*\|\"timezone\":\"[^\"]*\"' /tmp/groupscout-cadence-review.json"
docker exec groupscout_n8n sh -lc "grep -o 'guarantee_one_lead\\|delivery_mode\\|cadence_key\\|schedule_key\\|idempotency_key\\|notified_leads\\|delivery_status' /tmp/groupscout-cadence-review.json"
```

Expected export fields are `"active":true`, `"triggerAtDay":[0,1,2,3,4,5]`, `"triggerAtHour":9`, `"triggerAtMinute":0`, and `"timezone":"America/Vancouver"`. The second grep must show the guaranteed `/run` body plus `delivery_status`/`notified_leads` classification; if it still shows `JSON.stringify({})`, the live workflow is using the old non-guaranteed cadence body and must be re-imported from the tracked JSON.

### UI Verification

Open `http://localhost:5678`, then open **GroupScout Daily Lead Cadence (except Saturday)**.

Verify:

- the workflow toggle is **Active**;
- the schedule node runs every day except Saturday at `09:00` in `America/Vancouver`;
- **Trigger GroupScout Run** uses `POST` to `{{$env.GROUPSCOUT_API_BASE_URL}}/run` or the `http://groupscout:8080/run` fallback;
- the Authorization header uses `Bearer {{$env.GROUPSCOUT_API_TOKEN}}`;
- the JSON body includes `guarantee_one_lead`, `delivery_mode: "all_eligible"`, `cadence_key`, `schedule_key`, and `idempotency_key`, so `/run` uses idempotent cadence delivery for every eligible lead;
- both ops Slack HTTP nodes use `{{$env.GROUPSCOUT_OPS_SLACK_WEBHOOK_URL}}`;
- the success path is `Delivered?` to `Mark Delivered`, and failure/no-lead paths go to the ops Slack nodes.

### Backend Contract

The backend `/run` cadence mode returns JSON with `new_leads`, `notified_leads`, `delivery_status`, `delivered_lead_id`, `delivered_lead_ids`, `idempotency_key`, and `schedule_key`. The workflow treats `delivery_status == "sent"` or `"duplicate"` as delivered and treats `delivery_status == "no_eligible_lead"` as an operational no-lead outcome. Manual ad-hoc `/run` calls without a cadence key still use the normal multi-lead notification path.
