# GroupScout n8n Workflows

## Sunday/Tuesday/Thursday 6 PM Lead Cadence

Import `sunday-wednesday-lead-cadence.json` into n8n for the scheduled internal lead send. The filename and workflow id (`groupscout-sunday-wednesday-lead-cadence`) are retained for stability; the schedule now runs Sunday, Tuesday, and Thursday at 18:00 in `America/Vancouver`.

The workflow:

- runs Sunday, Tuesday, and Thursday at 18:00 in `America/Vancouver` (`triggerAtDay: [0,2,4]`);
- writes a cadence key such as `lead-cadence:2026-05-27:wednesday` in workflow static data and stops duplicate runs only after a cadence is marked delivered;
- calls `GET /health` before running the pipeline;
- calls `POST /run` with a bearer token and a cadence-delivery body containing `guarantee_one_lead`, `delivery_mode`, `cadence_key`, `schedule_key`, and `idempotency_key`;
- sends the backend's human-readable operational Slack message when health fails, the run fails, or the backend returns no notified leads;
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

Then open n8n at `http://localhost:5678`, confirm the workflow is inactive after import, set any missing env-backed values or credentials, run **Test Workflow**, and activate it only after the health and `/run` nodes return the expected status. CLI publish plus activation is:

```sh
docker exec groupscout_n8n n8n publish:workflow --id=groupscout-sunday-wednesday-lead-cadence
docker restart groupscout_n8n
docker exec groupscout_n8n n8n export:workflow --id=groupscout-sunday-wednesday-lead-cadence --output=/tmp/groupscout-cadence-review.json
docker exec groupscout_n8n sh -lc "grep -o '\"active\":[^,}]*\|\"triggerAtDay\":\[[^]]*\]\|\"triggerAtHour\":[0-9]*\|\"triggerAtMinute\":[0-9]*\|\"timezone\":\"[^\"]*\"' /tmp/groupscout-cadence-review.json"
docker exec groupscout_n8n sh -lc "grep -o 'guarantee_one_lead\\|delivery_mode\\|cadence_key\\|schedule_key\\|idempotency_key\\|notified_leads\\|delivery_status\\|message' /tmp/groupscout-cadence-review.json"
```

Expected export fields are `"active":true`, `"triggerAtDay":[0,2,4]`, `"triggerAtHour":18`, `"triggerAtMinute":0`, and `"timezone":"America/Vancouver"`. The second grep must show the guaranteed `/run` body plus `delivery_status`/`notified_leads`/`message` classification; if it still shows `JSON.stringify({})`, the live workflow is using the old non-guaranteed cadence body and must be re-imported from the tracked JSON.

`n8n update:workflow --active=true` still works in this environment, but n8n now marks it deprecated. Prefer `publish:workflow` and then restart the `groupscout_n8n` container so the schedule trigger re-registers.

### Change Timing

Use this when the cadence should run on different days or at a different time:

1. Edit `backend/docs/workflows/n8n/sunday-wednesday-lead-cadence.json`.
2. Update the schedule node `triggerAtDay`, `triggerAtHour`, and `triggerAtMinute`.
3. If the visible cadence meaning changed, also update the workflow `name`, the schedule node `name`, and the `connections` key that points at that schedule node.
4. Keep the filename and workflow `id` stable unless every downstream reference is being updated at the same time.
5. Copy the file into the live container, import it, publish it, restart n8n, and export it again to verify the live schedule fields:

```sh
docker cp backend/docs/workflows/n8n/sunday-wednesday-lead-cadence.json groupscout_n8n:/tmp/sunday-wednesday-lead-cadence.json
docker exec groupscout_n8n n8n import:workflow --input=/tmp/sunday-wednesday-lead-cadence.json
docker exec groupscout_n8n n8n publish:workflow --id=groupscout-sunday-wednesday-lead-cadence
docker restart groupscout_n8n
docker exec groupscout_n8n n8n export:workflow --id=groupscout-sunday-wednesday-lead-cadence --output=/tmp/groupscout-cadence-review.json
```

Import deactivates the workflow, so publishing and restarting are both required before the new schedule can fire.

### Trigger Now

For an immediate operator test, use **Test Workflow** in the n8n UI or the play button on the schedule node. Against the live container, that path is more reliable than `n8n execute`, which can fail while the running instance already owns its task-broker port.

If you need the same cadence payload outside the UI, call `POST /run` with the same guaranteed cadence body the workflow uses and a current cadence key.

### UI Verification

Open `http://localhost:5678`, then open **GroupScout Lead Cadence (Sun/Tue/Thu 6 PM)**.

Verify:

- the workflow toggle is **Active**;
- the schedule node runs Sunday, Tuesday, and Thursday at `18:00` in `America/Vancouver`;
- **Trigger GroupScout Run** uses `POST` to `{{$env.GROUPSCOUT_API_BASE_URL}}/run` or the `http://groupscout:8080/run` fallback;
- the Authorization header uses `Bearer {{$env.GROUPSCOUT_API_TOKEN}}`;
- the JSON body includes `guarantee_one_lead`, `delivery_mode: "all_eligible"`, `cadence_key`, `schedule_key`, and `idempotency_key`, so `/run` uses idempotent cadence delivery for every eligible lead;
- both ops Slack HTTP nodes use `{{$env.GROUPSCOUT_OPS_SLACK_WEBHOOK_URL}}`;
- the success path is `Delivered?` to `Mark Delivered`, and failure/no-lead paths go to the ops Slack nodes; the no-lead Slack body should use `$json.message` instead of rebuilding raw `run_ok`/`no_lead`/`status` flags.

### Backend Contract

The backend `/run` cadence mode returns JSON with `message`, `new_leads`, `notified_leads`, `delivery_status`, `delivered_lead_id`, `delivered_lead_ids`, `idempotency_key`, and `schedule_key`. The workflow treats `delivery_status == "sent"` or `"duplicate"` as delivered and treats `delivery_status == "no_eligible_lead"` as an operational no-lead outcome. The no-lead Slack branch posts `message` verbatim when present. Manual ad-hoc `/run` calls without a cadence key still use the normal multi-lead notification path.
