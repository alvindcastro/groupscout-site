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

### 4. Pushing External Leads (`/n8n/webhook`)

Use this to send leads from other n8n-connected sources (RSS, Web Scrapers, Google Sheets) into GroupScout.

#### Node: **HTTP Request**
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
    "PriorityScore": 90,
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
| `PriorityScore`| Number | No | Relevance score (0-100). |
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

### 6. Scheduling with n8n

You can use the **Schedule** node in n8n to automate GroupScout runs at specific times.

#### Example: Run every Sunday and Wednesday at 9:00 AM

1.  Add a **Schedule** node to your workflow.
2.  Set **Trigger Interval** to `Weeks`.
3.  **Days of the Week**: Select `Sunday` and `Wednesday`.
4.  **Time**: Set to `09:00`.
5.  Set the workflow timezone to the hotel/operator timezone, for example `America/Vancouver`.
6.  Connect this node to an **HTTP Request** node configured for the `/run` endpoint (as described in Section 3).

---

### 7. Sunday/Wednesday One-Lead Send

The current backend can run the pipeline and notify all newly collected leads, but "always send one lead" is stronger than "run the pipeline." A normal run can produce zero fresh leads because GroupScout deduplicates source records, filters low-value records, and marks notified leads so they do not repeat as new.

#### n8n-only MVP

Use this when the goal is a dependable internal lead prompt without changing backend code:

1.  **Schedule**: run every `Sunday` and `Wednesday` at `09:00` in `America/Vancouver`.
2.  **Preflight**: call `GET http://groupscout:8080/health`. Stop and notify operations if the database is unavailable.
3.  **Trigger collection**: call `POST http://groupscout:8080/run` with the `GroupScout API` bearer credential.
4.  **Branch on result**:
    - If `/run` succeeds and Slack receives leads, treat the cadence as delivered.
    - Current `/run` returns plain success text, not a structured lead count. To check whether the run found leads, inspect GroupScout logs for `sent leads to Slack`, `no new leads to notify`, or pipeline errors.
    - If logs show no new leads, send an operational Slack note such as "No qualified GroupScout lead was available for the Wednesday cadence."
5.  **Retry policy**: let n8n retry transient HTTP failures, but keep a workflow variable or data-store key such as `lead-cadence:YYYY-MM-DD:sunday` so a retry does not send duplicate operational messages.

This MVP is safe because n8n only uses server-to-server `API_TOKEN` credentials and GroupScout remains the source of truth. It does not guarantee a real lead when no eligible lead exists.

Tracked follow-up: `groupscout-site-yfl` owns exporting and smoke-testing the importable n8n workflow JSON for this MVP.

#### Robust guaranteed-lead design

For a true "send exactly one lead every Sunday and Wednesday" guarantee, add backend support instead of encoding selection rules in n8n:

- A machine-readable `/run` response with counts such as `new_leads`, `notified_leads`, `fallback_lead_id`, and delivery status.
- A `lead_delivery_log` table with `lead_id`, `channel`, `schedule_key`, `sent_at`, `idempotency_key`, and `result`.
- A fallback selector that can resurface the best eligible backlog lead when no fresh lead exists. Prefer high-score leads, exclude `contacted`, `won`, `lost`, and `dismissed`, and exclude leads delivered by the scheduled cadence in the last 14-30 days.
- A run lock, preferably a Postgres advisory lock or `pipeline_locks` row, so n8n retries and manual pipeline starts cannot race each other.
- UI/system visibility for the next scheduled send, last sent lead, failed cadence reason, retry count, and n8n execution link.

With those pieces in place, n8n should orchestrate the cadence while the backend owns selection, idempotency, delivery state, and auditability.

Tracked follow-up: `groupscout-site-fuc`.

---

### 8. Troubleshooting & Tips

#### Common Errors
- **401 Unauthorized**: Check your `API_TOKEN` in `.env` matches the `Bearer` token in n8n.
- **400 Bad Request**: Your JSON body is invalid or missing the `Title` field.
- **Connection Refused**:
    - From Compose n8n to Compose backend, use `http://groupscout:8080`.
    - From host n8n to a host backend, use `http://localhost:8080`.
    - From containerized n8n to a host backend, use `http://host.docker.internal:8080` on Docker Desktop.
    - Check if the GroupScout server is actually running (`docker compose ps` or `go run cmd/server/main.go`).
- **No lead sent on Sunday/Wednesday**: A successful `/run` can still produce zero fresh leads. For the MVP, send an operational "no qualified lead" notice; for the robust version, implement the fallback selector and delivery log above.
- **Duplicate lead sent after retry**: Add a cadence idempotency key in n8n now, and move that guard into a backend `lead_delivery_log` before enabling automatic fallback delivery.

#### Advanced Workflow Example
1.  **RSS Read**: Check for new construction news.
2.  **AI/LLM (n8n node)**: Summarize the article into structured JSON.
3.  **GroupScout Webhook**: Send the structured JSON to GroupScout.
4.  **Slack (GroupScout Internal)**: GroupScout automatically notifies your team.

---

### 9. Docker Network Note
If you are using the provided `docker-compose.yml`, both services share the same network. You can reach GroupScout from n8n using:
- **URL**: `http://groupscout:8080/run` (or `/n8n/webhook`, `/digest`)

Check n8n-to-backend connectivity from the running n8n container:

```sh
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
```

Expected output includes `"database":"ok"`.
