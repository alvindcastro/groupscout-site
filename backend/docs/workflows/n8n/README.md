# GroupScout n8n Workflows

## Sunday/Wednesday Lead Cadence

Import `sunday-wednesday-lead-cadence.json` into n8n for the current n8n-only MVP of the Sunday/Wednesday internal lead send.

The workflow:

- runs every Sunday and Wednesday at 09:00 in `America/Vancouver`;
- writes an idempotency key such as `lead-cadence:2026-05-27:wednesday` in workflow static data and stops duplicate runs only after a cadence is marked delivered;
- calls `GET /health` before running the pipeline;
- calls `POST /run` with a bearer token;
- sends an operational Slack message when health fails, the run fails, or the run output looks like a no-leads result;
- retries transient HTTP node failures up to three times;
- stops silently on duplicate cadence-key runs after delivery.

Required n8n environment variables:

| Variable | Purpose |
|---|---|
| `GROUPSCOUT_API_BASE_URL` | GroupScout base URL. Use `http://groupscout:8080` from Compose n8n to Compose backend. |
| `GROUPSCOUT_API_TOKEN` | Bearer token matching backend `API_TOKEN`. |
| `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL` | Slack incoming webhook for operational failures/no-leads notices. |

The JSON keeps secrets out of the repository. If these environment variables are not set, imported nodes show placeholder values that must be replaced before activation.

The tracked export includes a stable workflow `id` because the Docker-backed n8n smoke stack currently runs n8n `2.14.2`, whose CLI import requires a non-empty workflow id.

### Docker Import Smoke

From the coordination repo root:

```sh
docker cp backend/docs/workflows/n8n/sunday-wednesday-lead-cadence.json groupscout_n8n:/tmp/sunday-wednesday-lead-cadence.json
docker exec groupscout_n8n n8n import:workflow --input=/tmp/sunday-wednesday-lead-cadence.json
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
```

Expected health output includes `"database":"ok"`.

Then open n8n at `http://localhost:5678`, confirm the workflow is inactive after import, set any missing env-backed values or credentials, run **Test Workflow**, and activate it only after the health and `/run` nodes return the expected status.

### Known Limit

This workflow cannot prove a real lead was delivered because the current backend `/run` response is plain text and does not include structured lead counts. It uses a conservative text classifier and sends an operational Slack message when the response looks like no leads were available. Backend support for true exactly-one delivery remains tracked separately in `groupscout-site-fuc`.
