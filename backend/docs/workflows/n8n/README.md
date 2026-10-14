# GroupScout n8n Workflows

The tracked workflow JSON lives in the **backend source repo** at `deploy/config/n8n/workflows/`. Since IaC Phase 2 (`groupscout-site-6ah.3`, 2026-10-14), the Compose `n8n` service imports it automatically. This coordination folder keeps only the runbook. The JSON copy that used to sit here was removed so the two can't drift.

## How Workflows Are Deployed

The `n8n` service starts through `deploy/config/n8n/bootstrap.sh`, mounted read-only at `/opt/groupscout/n8n`. Before the n8n server starts, inside the same container, the script runs these steps for every `deploy/config/n8n/workflows/*.json`:

1. **Skip** the file when the workflow id already exists in n8n and the file's sha256 matches the last import. The checksums are kept in `/home/node/.n8n/groupscout-workflows.sha256` inside the `n8n_data` volume.
2. **Unpublish** the existing workflow when the file changed. n8n 2.43 refuses to let the CLI deactivate an active workflow during import.
3. **Import** the file, then **publish** it when the JSON says `"active": true`. `import:workflow --activeState=fromJson` only works in queue or multi-main mode, so the script publishes explicitly.
4. Hand over to the normal n8n entrypoint.

What this guarantees:

- **An empty `n8n_data` volume recovers by itself.** After the July 2026 Podman switch, an empty volume silently stopped the cadence. Now n8n comes back with every tracked workflow imported and active, with no `podman cp` or `import` steps.
- **The import never runs while n8n is running**, so it can't race the server for the SQLite database.
- **Edits made in the n8n UI survive restarts until the tracked file changes.** The next start after a file change replaces the live workflow with git's version. Export UI changes back into the JSON if you want to keep them.
- **A re-import resets the workflow's static data**, which includes the cadence keys used for dedupe. The schedule only fires at the scheduled time, so a restart doesn't cause a duplicate send.
- **A failing import stops the container before n8n starts.** `docker compose ps` then shows a restart loop instead of a quietly dead schedule. Check `docker compose logs n8n` for `[groupscout-n8n-bootstrap]` lines.

## Sunday/Tuesday/Thursday 6 PM Lead Cadence

File: `deploy/config/n8n/workflows/sunday-wednesday-lead-cadence.json`. The filename and workflow id (`groupscout-sunday-wednesday-lead-cadence`) are retained for stability. The schedule runs Sunday, Tuesday and Thursday at 18:00 in `America/Vancouver`.

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

The JSON keeps secrets out of the repository. If these environment variables are not set, the imported nodes show placeholder values.

This workflow calls `/run` for cadence delivery. For separate event-driven source pushes, use `POST /ingest` when GroupScout should enrich one raw project/event payload through `EnrichOne()`. Use `/n8n/webhook` only for pre-enriched lead-shaped payloads that should be inserted directly.

## Change A Workflow (Timing Or Anything Else)

From the backend source repo:

1. Edit `deploy/config/n8n/workflows/sunday-wednesday-lead-cadence.json`. For timing, change the schedule node's `triggerAtDay`, `triggerAtHour` and `triggerAtMinute`.
2. If the visible cadence meaning changed, also update the workflow `name`, the schedule node `name`, and the `connections` key that points at that schedule node.
3. Keep the filename, the workflow `id`, and `"active": true` unless every downstream reference changes at the same time.
4. Run `go test ./internal/smoke/`, which checks that tracked workflows parse and keep an id, a name and an `active` flag.
5. Restart n8n so the bootstrap picks up the change:

```sh
docker compose restart n8n            # or: podman compose restart n8n
docker compose logs n8n | grep groupscout-n8n-bootstrap
# expect: unpublishing before re-import … importing … publishing …
```

6. Verify the live schedule and `/run` body:

```sh
docker exec groupscout_n8n n8n list:workflow --active=true
docker exec groupscout_n8n n8n export:workflow --id=groupscout-sunday-wednesday-lead-cadence --output=/tmp/groupscout-cadence-review.json
docker exec groupscout_n8n sh -lc "grep -o '\"active\":[^,}]*\|\"triggerAtDay\":\[[^]]*\]\|\"triggerAtHour\":[0-9]*\|\"triggerAtMinute\":[0-9]*\|\"timezone\":\"[^\"]*\"' /tmp/groupscout-cadence-review.json"
docker exec groupscout_n8n sh -lc "grep -o 'guarantee_one_lead\\|delivery_mode\\|cadence_key\\|schedule_key\\|idempotency_key\\|notified_leads\\|delivery_status\\|message' /tmp/groupscout-cadence-review.json"
```

Expected export fields are `"active":true`, `"triggerAtDay":[0,2,4]`, `"triggerAtHour":18`, `"triggerAtMinute":0` and `"timezone":"America/Vancouver"`. The second grep must show the guaranteed `/run` body plus the `delivery_status`/`notified_leads`/`message` classification.

Check the environment without printing secrets:

```sh
docker exec groupscout_n8n sh -lc 'for k in N8N_BLOCK_ENV_ACCESS_IN_NODE GROUPSCOUT_API_BASE_URL GROUPSCOUT_API_TOKEN GROUPSCOUT_OPS_SLACK_WEBHOOK_URL; do v=$(printenv "$k"); if [ -n "$v" ]; then echo "$k=SET"; else echo "$k=MISSING"; fi; done'
```

The n8n image has `node` but no `wget` or `curl`. Test backend reachability from n8n with `node -e "fetch('http://groupscout:8080/health').then(r=>r.text()).then(console.log)"`.

## Importing Outside The Compose Stack

For an n8n that doesn't use the backend Compose file (n8n Cloud, `npm install -g n8n`, another host), import the backend repo's JSON by hand. On n8n 2.43 and later:

```sh
n8n unpublish:workflow --id=groupscout-sunday-wednesday-lead-cadence   # only if it already exists
n8n import:workflow --input=sunday-wednesday-lead-cadence.json
n8n publish:workflow --id=groupscout-sunday-wednesday-lead-cadence
# then restart n8n so the schedule trigger registers
```

`n8n update:workflow --active=true` is deprecated. Use `publish:workflow`.
