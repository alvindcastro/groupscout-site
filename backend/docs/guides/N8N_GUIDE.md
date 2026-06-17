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

### 7. Sunday/Wednesday Guaranteed Lead Send

The Sunday/Wednesday cadence uses **guaranteed delivery mode** — the backend finds and delivers the single best available lead from the current run or, if nothing new scored above zero, falls back to the highest-priority undelivered `notified` lead from the backlog.

#### n8n workflow

1.  **Schedule**: run every `Sunday` and `Wednesday` at `09:00` in `America/Vancouver`.
2.  **Preflight**: call `GET http://groupscout:8080/health`. Stop and notify operations if the database is unavailable.
3.  **Trigger collection and delivery**: call `POST http://groupscout:8080/run` with the `GroupScout API` bearer credential and a JSON body that includes the cadence key and guaranteed delivery flag:
    ```json
    {
      "guarantee_one_lead": true,
      "delivery_mode": "exactly_one",
      "cadence_key": "lead-cadence:YYYY-MM-DD:wednesday",
      "idempotency_key": "lead-cadence:YYYY-MM-DD:wednesday"
    }
    ```
    The tracked n8n workflow computes the cadence key dynamically via the **Build Cadence Key** code node and injects it into the body expression.
4.  **Branch on result**:
    - `delivery_status == "sent"`: one Slack lead was delivered; mark the cadence key as done.
    - `delivery_status == "no_eligible_lead"`: no lead above the score threshold exists anywhere in the system; send an operational Slack note.
    - non-2xx response: send an operational Slack note and retry later.
5.  **Idempotency**: the backend writes a `lead_deliveries` row keyed on `idempotency_key`; duplicate cadence fires (n8n retries, manual triggers) are safely skipped.

**Backend defensive guard (2026-06-17)**: as of commit `038c08b`, the backend treats any `/run` request that includes a non-empty `cadence_key` or `schedule_key` as a guaranteed-delivery run, regardless of whether `guarantee_one_lead` parsed correctly. This defends against n8n expression evaluation failures that produce an invalid JSON body and silently default `GuaranteeOneLead` to false.

Importable workflow asset: [`../workflows/n8n/sunday-wednesday-lead-cadence.json`](../workflows/n8n/sunday-wednesday-lead-cadence.json). Import and Docker smoke notes live in [`../workflows/n8n/README.md`](../workflows/n8n/README.md).

For the provided Docker Compose stack, n8n reads the tracked workflow expressions from container environment variables. Recreate n8n after env changes with `docker compose up -d n8n`, then verify `N8N_BLOCK_ENV_ACCESS_IN_NODE`, `GROUPSCOUT_API_BASE_URL`, `GROUPSCOUT_API_TOKEN`, and `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL` are all present before activating the workflow.

When validating an existing imported workflow, export it and confirm the `/run` HTTP node body expression evaluates to a JSON object containing `guarantee_one_lead: true`, `delivery_mode: "exactly_one"`, and a cadence key. The schedule must be active for Sunday/Wednesday 09:00 `America/Vancouver`.

#### Local UI checklist

Use this when an operator wants to visually confirm the workflow before leaving it to the schedule:

1. Open `http://localhost:5678`.
2. Sign in with the local n8n owner account.
3. Open **GroupScout Sunday Wednesday Lead Cadence**.
4. Confirm the top-right workflow toggle is **Active**.
5. Confirm the graph contains schedule, cadence-key, duplicate guard, health preflight, `/run` trigger, run classifier, delivered marker, and ops Slack failure/no-lead branches.
6. Open **Trigger GroupScout Run** and confirm the body expression includes `guarantee_one_lead: true` and `cadence_key`.
7. Open each ops Slack node and confirm the URL is environment-backed, not the placeholder webhook URL.

#### Backend delivery guarantee

The backend guarantee uses:

- A machine-readable `/run` response with `new_leads`, `notified_leads`, `delivered_lead_id`, `idempotency_key`, `schedule_key`, and `delivery_status`.
- A `lead_deliveries` table with `lead_id`, `channel`, `schedule_key`, `sent_at`, `idempotency_key`, `status`, and `result`.
- A fallback selector that prefers fresh `new` leads, then resurfaces the best eligible `notified` backlog lead that has not already been delivered by the cadence.
- A `delivery_locks` row so n8n retries and manual pipeline starts cannot race each other.

Future UI/system visibility for next scheduled send, last notified count, failed cadence reason, retry count, and n8n execution link remains part of the operator UI/API roadmap.

The exactly-one backend guarantee is the default for the scheduled n8n cadence. Manual ad-hoc `/run` calls without a `cadence_key` use the non-guaranteed path and fan out all current `new` leads in one Slack burst.

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
- **No lead sent on Sunday/Wednesday (`delivery_status: no_eligible_lead`)**: No lead in the DB has `status IN ('new','notified')` with a score above zero that hasn't already been cadence-delivered. This is a genuine empty-pool event. Check recent collector logs for parse failures or score-zero runs. Collector issues (VCC 404, creativebc parse error) shrink the pool.
- **Cadence runs in non-guaranteed mode (`no new leads to notify` in logs)**: The backend logged this message on the non-guaranteed code path, meaning `cadence_key`/`schedule_key` was empty AND `guarantee_one_lead` was false. Confirm the `/run` HTTP node body expression includes `cadence_key`. As of 2026-06-17, any request with a non-empty `cadence_key` or `schedule_key` is automatically treated as guaranteed.
- **Cadence workflow never sends**: Confirm the imported workflow is active, the n8n container was restarted after activation, and the live export still shows `triggerAtDay:[0,3]`, `triggerAtHour:9`, `triggerAtMinute:0`, and `timezone:"America/Vancouver"`.
- **Cadence sends only one lead**: This is the intended behavior — the Sunday/Wednesday cadence uses `guarantee_one_lead: true` and delivers exactly one high-priority lead per run. If you want all new leads in a single Slack burst, call `/run` with an empty body manually (omit the cadence key so non-guaranteed mode is used).
- **Cannot sign in to local n8n**: Use `n8n user-management:reset` with the container stopped. If sign-in still shows an existing owner account, follow the local SQLite owner recovery steps in [Troubleshooting](./TROUBLESHOOTING.md#6-recover-local-n8n-sign-in).
- **Duplicate scheduled send after retry**: The tracked workflow stores the cadence key after a successful delivery and stops duplicate same-cadence workflow runs. The backend also guards via `lead_deliveries` idempotency key — a second request with the same `idempotency_key` is a no-op if the first delivered successfully.

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
