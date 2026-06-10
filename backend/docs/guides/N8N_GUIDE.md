### n8n Integration Guide for GroupScout

This guide provides comprehensive instructions for integrating n8n with GroupScout. It covers triggering the internal pipeline, pushing external leads, managing automated digests, and coordinating a Sunday/Wednesday internal lead send.

---

### 1. Prerequisites

Before you begin, ensure you have the following:

- **GroupScout Server**: Running and accessible (via `go run cmd/server/main.go` or Docker).
- **API Token**: A user-defined secret for authentication.
    - **How to generate**: Run `openssl rand -hex 32`.
    - **Set it**: In your `.env` file: `API_TOKEN=your_generated_token`.
- **n8n Instance**:
    - **Self-Hosted**: Use the provided `docker-compose.yml`. n8n will be at `http://localhost:5678`.
    - **Cloud/External**: Ensure it can reach your server's IP. If GroupScout is in Docker, use `http://host.docker.internal:8080` (Mac/Win) or the container's IP.

---

### 2. Setting up Authentication in n8n

To simplify your workflows, create a "Header Auth" credential in n8n:

1.  In n8n, go to **Credentials** > **Add Credential**.
2.  Search for **Header Auth**.
3.  **Name**: `GroupScout API`.
4.  **Header Name**: `Authorization`.
5.  **Value**: `Bearer <YOUR_API_TOKEN>` (Replace `<YOUR_API_TOKEN>` with the token from your `.env`).

---

### 3. Triggering the Full Pipeline (`/run`)

Use this to start the GroupScout collection, enrichment, and notification process.

#### Node: **HTTP Request**
- **Method**: `POST`
- **URL**: `http://<groupscout-host>:8080/run`
- **Authentication**: `Predefined Credential` (Select your `GroupScout API` credential).
- **Body Parameters** (Optional):
    - **Mode**: `JSON`
    - **Property**: `bcbid_raw_input`
    - **Value**: `<Raw text for manual BC Bid processing>`

---

### 4. Pushing External Leads (`/ingest` or `/n8n/webhook`)

Use this to send leads from other n8n-connected sources (RSS, web scrapers, Google Sheets) into GroupScout.

Use `POST /ingest` when the upstream source has raw or lightly normalized project/event data and GroupScout should run the same dedup, audit, scoring, enrichment, and lead storage path used by collectors. The backend branch `task/event-driven-ingest` implements this path with `EnrichOne()`.

Keep `/n8n/webhook` for pre-enriched lead-shaped payloads. It direct-inserts a `Lead` and optionally notifies Slack; it does not rerun enrichment or create a raw-project audit record.

#### Node: **HTTP Request**
- **Method**: `POST`
- **URL**: `http://<groupscout-host>:8080/ingest`
- **Authentication**: `Predefined Credential`.
- **Body Mode**: `JSON`
- **JSON Payload Example**:
  ```json
  {
    "source": "n8n-rss",
    "external_id": "evt-123",
    "title": "Richmond warehouse infrastructure",
    "location": "Richmond BC",
    "project_value": 12000000,
    "description": "Civil warehouse expansion with likely travelling crews.",
    "source_url": "https://example.com/project",
    "raw_data": "{\"title\":\"Richmond warehouse infrastructure\"}",
    "raw_type": "application/json",
    "metadata": {
      "applicant": "Example Applicant"
    }
  }
  ```

The response is `201` with `{"status":"created","inserted":true,...}` when a lead is inserted, or `200` with `{"status":"duplicate","inserted":false,...}` when the normalized payload hash has already been seen.

#### Direct Lead Insert Fallback: `/n8n/webhook`
Use this only when n8n has already produced a pre-enriched lead-shaped payload. If GroupScout should score and enrich one raw source item, use `POST /ingest` instead.

Priority scores are stored and rendered on the Slack/internal `0-10` scale. New workflows should send `PriorityScore` as `0` through `10`; older n8n workflows that still send percent-style `0-100` values are normalized before storage and Slack delivery.

- **Method**: `POST`
- **URL**: `http://<groupscout-host>:8080/n8n/webhook`
- **Authentication**: `Predefined Credential`.
- **Body Mode**: `JSON`
- **JSON Payload Example**:
  ```json
  {
    "Title": "New Tech Hub Construction",
    "Location": "Surrey, BC",
    "Source": "Custom RSS Scraper",
    "ProjectValue": 25000000,
    "ProjectType": "Commercial",
    "PriorityScore": 9,
    "PriorityReason": "High value project near transit.",
    "SourceURL": "https://example.com/project",
    "GeneralContractor": "Major Build Inc.",
    "EstimatedCrewSize": 50,
    "OutOfTownCrewLikely": false
  }
  ```

#### Detailed Schema for `/n8n/webhook`:
| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `Title` | String | **Yes** | Name of the project or lead. |
| `Location` | String | No | City, region, or specific address. |
| `Source` | String | No | Source name (defaults to "n8n"). |
| `ProjectValue` | Number | No | Estimated total project value (Integer/Long). |
| `ProjectType` | String | No | e.g., "Commercial", "Hotel", "Residential". |
| `PriorityScore`| Number | No | Relevance score on the Slack/internal 0-10 scale. Legacy 0-100 inputs are normalized before storage and Slack delivery. |
| `PriorityReason`| String | No | Brief explanation for the score. |
| `SourceURL` | String | No | Direct link to the source document/page. |
| `GeneralContractor`| String | No | Name of the GC. |
| `Applicant` | String | No | Permit applicant name/contact. |
| `Contractor` | String | No | Trade contractor name/contact. |
| `EstimatedCrewSize` | Number | No | Estimated workers needed. |
| `EstimatedDurationMonths` | Number | No | Estimated project length. |
| `OutOfTownCrewLikely` | Boolean | No | `true` if crew likely from outside local area. |
| `SuggestedOutreachTiming` | String | No | e.g., "Immediate", "In 3 months". |
| `Notes` | String | No | Any additional context. |

---

### 5. Automated Weekly Digest (`/digest`)

Trigger a summary email of all "new" or "notified" leads from the last 7 days.

#### Node: **HTTP Request**
- **Method**: `POST`
- **URL**: `http://<groupscout-host>:8080/digest?to=sales@yourcompany.com`
- **Authentication**: `Predefined Credential`.

---

### 6. Basic Scheduling with n8n

You can use the **Schedule** node in n8n to automate GroupScout runs at specific times. The tracked Sunday/Wednesday workflow uses this plain `/run` mode so Slack receives the same multi-lead digest as `go run cmd/server/main.go --run-once`.

#### Example: Run every Sunday and Wednesday at 9:00 AM

1.  Add a **Schedule** node to your workflow.
2.  Set **Trigger Interval** to `Weeks`.
3.  **Days of the Week**: Select `Sunday` and `Wednesday`.
4.  **Time**: Set to `09:00`.
5.  Set the workflow timezone to the hotel/operator timezone, for example `America/Vancouver`.
6.  Connect this node to an **HTTP Request** node configured for the `/run` endpoint (as described in Section 3).

---

### 7. Sunday/Wednesday Normal Lead Send

The backend can run the pipeline and notify all newly collected leads. The Sunday/Wednesday n8n cadence should call normal `/run` so it sends all current `new` leads, then marks them `notified`, matching the command-line `--run-once` path.

#### n8n workflow

Use this when the goal is a dependable scheduled version of the normal pipeline:

1.  **Schedule**: run every `Sunday` and `Wednesday` at `09:00` in `America/Vancouver`.
2.  **Preflight**: call `GET http://groupscout:8080/health`. Stop and notify operations if the database is unavailable.
3.  **Trigger collection and delivery**: call `POST http://groupscout:8080/run` with the `GroupScout API` bearer credential and an empty JSON body:
    ```json
    {}
    ```
4.  **Branch on result**:
    - `notified_leads > 0`: one Slack digest was sent containing all current new leads.
    - `notified_leads == 0`: no current leads were available for Slack; send an operational Slack note.
    - non-2xx response: send an operational Slack note and retry later.
5.  **Retry policy**: let n8n retry transient HTTP failures, but keep a workflow variable or data-store key such as `lead-cadence:YYYY-MM-DD:sunday` so n8n can stop early after a successful scheduled send.

This flow is safe because n8n only uses server-to-server `API_TOKEN` credentials and GroupScout remains the source of truth. It cannot fabricate a lead when no eligible lead exists; a zero-notification result goes to the ops branch.

Importable workflow asset: [`../workflows/n8n/sunday-wednesday-lead-cadence.json`](../workflows/n8n/sunday-wednesday-lead-cadence.json). Import and Docker smoke notes live in [`../workflows/n8n/README.md`](../workflows/n8n/README.md).

For the provided Docker Compose stack, n8n reads the tracked workflow expressions from container environment variables. Recreate n8n after env changes with `docker compose up -d n8n`, then verify `N8N_BLOCK_ENV_ACCESS_IN_NODE`, `GROUPSCOUT_API_BASE_URL`, `GROUPSCOUT_API_TOKEN`, and `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL` are all present before activating the workflow.

When validating an existing imported workflow, export it and check for both schedule and request body. The schedule must be active for Sunday/Wednesday 09:00 `America/Vancouver`, and the `/run` node must use an empty `{}` JSON body. If the live export still contains `guarantee_one_lead`, `delivery_mode`, or `idempotency_key`, re-import the tracked JSON; otherwise n8n will keep using the old exactly-one cadence path.

#### Local UI checklist

Use this when an operator wants to visually confirm the workflow before leaving it to the schedule:

1. Open `http://localhost:5678`.
2. Sign in with the local n8n owner account.
3. Open **GroupScout Sunday Wednesday Lead Cadence**.
4. Confirm the top-right workflow toggle is **Active**.
5. Confirm the graph contains schedule, cadence-key, duplicate guard, health preflight, normal `/run`, run classifier, delivered marker, and ops Slack failure/no-lead branches.
6. Open **Trigger GroupScout Run** and confirm the body is `{}` so it does not request exactly-one delivery.
7. Open each ops Slack node and confirm the URL is environment-backed, not the placeholder webhook URL.

#### Backend delivery guarantee

The backend guarantee uses:

- A machine-readable `/run` response with `new_leads`, `notified_leads`, `delivered_lead_id`, `idempotency_key`, `schedule_key`, and `delivery_status`.
- A `lead_deliveries` table with `lead_id`, `channel`, `schedule_key`, `sent_at`, `idempotency_key`, `status`, and `result`.
- A fallback selector that prefers fresh `new` leads, then resurfaces the best eligible `notified` backlog lead that has not already been delivered by the cadence.
- A `delivery_locks` row so n8n retries and manual pipeline starts cannot race each other.

Future UI/system visibility for next scheduled send, last notified count, failed cadence reason, retry count, and n8n execution link remains part of the operator UI/API roadmap.

The older exactly-one backend guarantee remains implemented under `groupscout-site-fuc` for manual/special-purpose cadence tests, but it is no longer the default scheduled n8n workflow.

---

### 8. Troubleshooting & Tips

#### Common Errors
- **401 Unauthorized**: Check your `API_TOKEN` in `.env` matches the `Bearer` token in n8n.
- **400 Bad Request**: Your JSON body is invalid or missing endpoint-specific required fields.
  - For `/n8n/webhook`, include the lead-shaped `Title` field.
  - For `/ingest`, include at least one of `title`, `description`, or `raw_data`; malformed JSON also returns `400`.
- **Connection Refused**:
    - From Compose n8n to Compose backend, use `http://groupscout:8080`.
    - From host n8n to a host backend, use `http://localhost:8080`.
    - From containerized n8n to a host backend, use `http://host.docker.internal:8080` on Docker Desktop.
    - Check if the GroupScout server is actually running (`docker compose ps` or `go run cmd/server/main.go`).
- **No lead sent on Sunday/Wednesday**: Check `/run` JSON. `notified_leads:0` means no fresh `new` leads were available for Slack; send an operational "no qualified lead" notice.
- **Cadence workflow never sends**: Confirm the imported workflow is active, the n8n container was restarted after activation, and the live export still shows `triggerAtDay:[0,3]`, `triggerAtHour:9`, `triggerAtMinute:0`, and `timezone:"America/Vancouver"`.
- **Cadence sends only one lead**: Export the workflow and confirm the `/run` HTTP node no longer includes `guarantee_one_lead`, `delivery_mode`, or `idempotency_key`. Re-import the tracked workflow if those fields are present.
- **Cannot sign in to local n8n**: Use `n8n user-management:reset` with the container stopped. If sign-in still shows an existing owner account, follow the local SQLite owner recovery steps in [Troubleshooting](./TROUBLESHOOTING.md#6-recover-local-n8n-sign-in).
- **Duplicate scheduled send after retry**: The tracked workflow stores the cadence key after a successful delivery and stops duplicate same-cadence workflow runs. Backend exactly-one idempotency is only used if you intentionally call the guarantee mode.

#### Advanced Workflow Example
1.  **RSS Read**: Check for new construction news.
2.  **AI/LLM (n8n node)**: Summarize the article into structured JSON.
3.  **GroupScout Webhook**: Send the structured JSON to GroupScout.
4.  **Slack (GroupScout Internal)**: GroupScout automatically notifies your team.

---

### 9. Docker Network Note
If you are using the provided `docker-compose.yml`, both services share the same network. You can reach GroupScout from n8n using:
- **URL**: `http://groupscout:8080/run` (or `/ingest`, `/n8n/webhook`, `/digest`)

Check n8n-to-backend connectivity from the running n8n container:

```sh
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
```

Expected output includes `"database":"ok"`.
