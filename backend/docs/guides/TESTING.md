### GroupScout Testing Strategy

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/backend/docs/guides/TESTING.md`. Run backend test commands from `/mnt/c/Users/alvin/GolandProjects/groupscout`; run UI test commands from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

The GroupScout testing infrastructure ensures the reliability of the lead collection, enrichment, and notification pipeline. It focuses on unit testing, data parsing verification, and end-to-end integration checks.

### Docker And UI Smoke

Use [BACKEND_FOR_UI_TESTING.md](../planning/ui/BACKEND_FOR_UI_TESTING.md) for local backend startup notes and [Backend And Frontend Docker E2E](../planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md) for the backend plus external UI smoke runbook. This guide keeps test commands and links to canonical runtime docs instead of duplicating Docker lifecycles.

### 1. Automated Go Tests (Makefile)

The project includes standard Go unit and integration tests. The recommended way to run them is via the `Makefile`.

#### Run All Tests
```bash
make test
```

#### Run Specific Package Tests
```bash
go test -v ./internal/enrichment/...
```

#### Planned UI API Contract Tests
```bash
go test ./internal/storage
go test ./cmd/server
```

Status reconciliation, 2026-05-25: the inspected backend source snapshot does not currently include the planned UI API handlers or admin auth/session tests. The package commands above are still useful for current storage/server coverage, but they do not prove `/api/leads`, `/api/leads/{id}`, `PATCH /api/leads/{id}`, or authenticated `/api/leads/{id}/raw` behavior until `groupscout-site-eqm` lands.

Phase 36 and 37 should extend the same package tests with outreach logging, lead actions, pipeline run persistence, stats, `/api/system`, and the read-only `/api/alerts` compatibility route.

The planned admin-login completion tests should also live in `cmd/server`: invalid setup-token login, file-backed setup-token rotation, bearer session-token access, `/api/auth/me`, session expiry, `/api/auth/logout` revocation, and legacy `/leads/{id}/raw` auth coverage.

#### Planned Admin Auth Token Flow Smoke

The current backend source snapshot does not expose `/api/auth/*`; run this smoke only after the admin auth/session implementation lands.

Local backend:

```bash
cd /mnt/c/Users/alvin/GolandProjects/groupscout
curl -i http://localhost:8080/api/auth/status
SETUP_TOKEN="$(cat data/admin-setup-token)"
curl -i -c /tmp/groupscout-admin.cookies \
  -H "Content-Type: application/json" \
  -d "{\"token\":\"${SETUP_TOKEN}\"}" \
  http://localhost:8080/api/auth/login
curl -i -b /tmp/groupscout-admin.cookies http://localhost:8080/api/auth/me
curl -i -b /tmp/groupscout-admin.cookies -c /tmp/groupscout-admin.cookies \
  -X POST http://localhost:8080/api/auth/logout
```

Docker backend:

```bash
SETUP_TOKEN="$(docker exec groupscout_app sh -lc 'cat data/admin-setup-token')"
curl -i -c /tmp/groupscout-admin.cookies \
  -H "Content-Type: application/json" \
  -d "{\"token\":\"${SETUP_TOKEN}\"}" \
  http://localhost:3001/api/auth/login
curl -i -b /tmp/groupscout-admin.cookies http://localhost:3001/api/auth/me
```

File-backed setup tokens rotate after successful login. If a setup token is rejected after a successful login, read `data/admin-setup-token` again. Env-backed `ADMIN_SETUP_TOKEN` values do not rotate automatically.

#### Planned UI Docker Smoke Contract Tests
```bash
go test ./internal/smoke
```

Status reconciliation, 2026-05-25: the inspected backend source snapshot does not currently include `internal/smoke` or `make smoke-ui-docker-e2e`. Use this once the Phase 38 smoke contract is restored or merged in `groupscout-site-crz`. To run the full Docker smoke against the external UI repo after that:

```bash
make smoke-ui-docker-e2e
```

#### Run Alertd Tests
```bash
go test ./cmd/alertd/... ./internal/alert/...
```

#### Planned AI Quality EvalOps Tests
```bash
go test -v ./internal/evalops
```

#### Planned GQ5 Draft-Case Helper Tests
```bash
go test -v ./internal/evalops -run 'TestDraftCasesFromReviewSamples|TestWriteDraftCasesJSONL'
```

Status reconciliation, 2026-05-25: the inspected backend source snapshot does not currently include `internal/evalops` or the EvalOps Makefile targets. Treat this section as branch-history/planned guidance until the EvalOps runtime is restored or merged in `groupscout-site-crz`.

These tests verify that redacted review samples become review-only draft cases with trace IDs, TODO expected fields, deterministic duplicate IDs, and a loader rejection until human review is complete.

#### Run Offline AI Quality Reports
```bash
make eval-quality
```

This writes deterministic JSON, Markdown, and JUnit artifacts under `build/evals/` without live provider keys.

#### Run The AI Quality Release Gate
```bash
make eval-gate
```

The default thresholds in `evals/promptfoo/thresholds.yaml` block critical or release-blocking failures and allow warning-only reports unless `warnings_as_errors` is enabled.

#### Run Promptfoo Against The Local Go Target
```bash
make eval-target
promptfoo eval -c evals/promptfoo/groupscout.yaml
```

Promptfoo calls `http://localhost:18080/eval/run` and passes fixture `case_id` variables to the local Go HTTP target. This path is CI-safe by default because it uses synthetic fixtures and deterministic scorers only.

---

### 2. Ollama Integration Testing

Since GroupScout relies on local LLMs for extraction and scoring, you need to verify that Ollama is correctly configured and the necessary models are available.

#### Using the Test Script
We provide a helper script to check Ollama connectivity, model availability, and basic inference:
```bash
./scripts/test-ollama.sh
```

#### Manual Verification
1.  **Check if Ollama is running**: `curl http://localhost:11434/api/tags`
2.  **Pull required base models**:
    ```bash
    make ollama-pull
    # OR
    ollama pull mistral
    ollama pull llama3.1:8b
    ollama pull phi3:mini
    ```
3.  **Push persona Modelfiles**:
    This creates the specific personas (like `permit_extractor`) used by GroupScout:
    ```bash
    make ollama-push
    # OR
    go run cmd/server/main.go ollama push-models
    ```

---

### 3. Manual Verification & Tools
Several utility scripts are provided for manual verification:
- `scripts/test-ollama.sh`: Verifies Ollama connectivity and models.
- `cmd/test_sentry/main.go`: Verifies Sentry connectivity and error reporting.
- `check_db.go`: A quick script to inspect the contents of the SQLite `groupscout.db`.
- `/run` endpoint: Allows triggering a full pipeline execution manually via HTTP.

### 4. Integration Testing
Integration tests are available for the storage layer and require a running database instance.

- **SQLite**: Standard tests run on SQLite by default.
- **Postgres**: Integration tests for Postgres are gated by the `TEST_POSTGRES_URL` environment variable and the `integration` build tag.

Start or confirm the Postgres test container:

```bash
cd /mnt/c/Users/alvin/GolandProjects/groupscout
docker compose up -d postgres
docker compose ps postgres
```

**Run Postgres integration tests:**
```powershell
$env:TEST_POSTGRES_URL="postgres://groupscout:groupscout@localhost:5432/groupscout?sslmode=disable"
go test -v -tags integration ./internal/storage/...
```

These tests verify:
- Dynamic driver selection (`DriverName`).
- SQL placeholder rebinding (`Rebind`).
- Versioned migrations (`Migrate`) using `golang-migrate`.
- Native Postgres type handling (e.g., `BOOLEAN`, `JSONB`).
- Vector similarity search (`EmbeddingStore`) using `pgvector` operators (e.g., `<=>`).
- CRUD operations and idempotency.

**Run embedding-specific unit tests (SQLite/Go-native):**
```powershell
go test -v ./internal/storage/ -run EmbeddingStore
```

**Trigger the pipeline manually (Docker):**
```bash
curl -X POST http://localhost:8080/run \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

**Check what happened after a run:**
```bash
docker compose logs groupscout --tail=50
```

**Follow logs in real time during a run:**
```bash
docker compose logs -f groupscout
```

### Backend Plus UI Docker Smoke Checks

Use this when the Phase 38 smoke target exists and you want to verify that the backend Compose stack and the separate UI Docker image can run together:

```bash
make smoke-ui-docker-e2e
```

The canonical manual runbook is [Backend And Frontend Docker E2E](../planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md).

### 5. Collector Test Pattern
When adding a new collector, follow the pattern used in `internal/collector/richmond_test.go`:
1. Define a `sampleLines` or `sampleHTML` variable with representative raw data.
2. Write tests for individual parsing helper functions (e.g., `parseDate`, `parseValue`).
3. Write a high-level test for the `Collect` or `process` function using a mock implementation of the source if possible.

### 6. CI/CD & Reliability
- **Deduplication**: Tests in `leads_test.go` (if implemented) or during integration ensure that the same lead is not processed multiple times.
- **Error Handling**: The Sentry integration (Phase 8.2) captures runtime exceptions, ensuring that transient failures in collectors are visible in the observability dashboard.

### 7. API Testing

For detailed instructions on testing API endpoints via `curl`, Bruno, or Swagger, see the [API Testing section](#8-api-testing-details) below.

### 8. API Testing Details

#### 8.1 Prerequisite: API Token
Most POST endpoints require a Bearer Token for authentication. This token is defined by the `API_TOKEN` environment variable in your `.env` file. If `API_TOKEN` is not set, the server will allow all requests (insecure mode, intended for local development only).

#### 8.2 Testing with `curl`

**Health Check (Port 8080)**
```bash
curl -i http://localhost:8080/health
```

**Manual Pipeline Run**
```bash
curl -i -X POST \
  -H "Authorization: Bearer your_api_token" \
  -H "Content-Type: application/json" \
  -d '{"bcbid_raw_input": ""}' \
  http://localhost:8080/run
```

**Guaranteed Sunday/Wednesday Delivery Run**
```bash
curl -i -X POST \
  -H "Authorization: Bearer your_api_token" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: lead-cadence:2026-05-27:wednesday" \
  -d '{"guarantee_one_lead":true,"delivery_mode":"exactly_one","cadence_key":"lead-cadence:2026-05-27:wednesday","idempotency_key":"lead-cadence:2026-05-27:wednesday"}' \
  http://localhost:8080/run
```

Expected `delivery_status` values are `sent`, `duplicate`, `no_eligible_lead`, or `locked`.

That curl proves the backend guarantee path only. To prove the n8n scheduled send path, also verify the live n8n container has its environment, can reach the backend, and has the imported cadence workflow activated:

```bash
docker exec groupscout_n8n sh -lc 'for k in N8N_BLOCK_ENV_ACCESS_IN_NODE GROUPSCOUT_API_BASE_URL GROUPSCOUT_API_TOKEN GROUPSCOUT_OPS_SLACK_WEBHOOK_URL; do v=$(printenv "$k"); if [ -n "$v" ]; then echo "$k=SET"; else echo "$k=MISSING"; fi; done'
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
docker exec groupscout_n8n n8n export:workflow --id=groupscout-sunday-wednesday-lead-cadence --output=/tmp/groupscout-cadence-review.json
docker exec groupscout_n8n sh -lc "grep -o '\"active\":[^,}]*\|\"triggerAtDay\":\[[^]]*\]\|\"triggerAtHour\":[0-9]*\|\"triggerAtMinute\":[0-9]*\|\"timezone\":\"[^\"]*\"' /tmp/groupscout-cadence-review.json"
docker exec groupscout_n8n sh -lc "grep -o 'guarantee_one_lead\|delivery_mode\|idempotency_key' /tmp/groupscout-cadence-review.json"
```

Expected n8n smoke result: every env check prints `SET`, health includes `"database":"ok"`, and the workflow export shows `"active":true`, Sunday/Wednesday days `[0,3]`, hour `9`, minute `0`, timezone `America/Vancouver`, and the guaranteed-cadence body fields.

**Trigger Weekly Digest**
```bash
curl -i -X POST \
  -H "Authorization: Bearer your_api_token" \
  "http://localhost:8080/digest?to=alvin@groupscout.ai"
```

**Raw Project/Event Ingest**
```bash
curl -i -X POST \
  -H "Authorization: Bearer your_api_token" \
  -H "Content-Type: application/json" \
  -d '{"source":"curl_test","title":"Richmond warehouse infrastructure","location":"Richmond BC","project_value":12000000,"description":"Civil warehouse expansion with likely travelling crews."}' \
  http://localhost:8080/ingest
```

Expected: `201` with `{"status":"created","inserted":true,...}` for a newly inserted lead, or `200` with `{"status":"duplicate","inserted":false,...}` when the normalized payload hash already exists. Postgres integration follow-up for the enrichment raw-input FK path is tracked by `groupscout-site-wda`.

**n8n Webhook Simulation**
```bash
curl -i -X POST \
  -H "Authorization: Bearer your_api_token" \
  -H "Content-Type: application/json" \
  -d '{
    "Source": "curl_test",
    "Title": "Simulated Lead",
    "Location": "Vancouver, BC",
    "ProjectValue": 500000,
    "PriorityScore": 9
  }' \
  http://localhost:8080/n8n/webhook
```

Expected: HTTP `201`. The lead is inserted directly and Slack delivery is attempted immediately. `PriorityScore` should be sent as `0-10`; legacy `0-100` values are normalized before storage and notification. To verify the regression fix for impossible Slack scores, send `PriorityScore:98`, then confirm the stored and notified score is `10`:

```bash
docker exec groupscout_postgres psql -U groupscout -d groupscout \
  -c "SELECT title, priority_score FROM leads WHERE source = 'curl_test' ORDER BY created_at DESC LIMIT 5;"
```

Slack and email display code also clamps scores defensively, so stored legacy rows cannot render as `98/10`.

**Slack Inventory Update (Port 8081)**
```bash
curl -i -X POST \
  -d "text=50" \
  http://localhost:8081/slack/inventory
```

#### 8.3 Testing with Bruno (Recommended)
[Bruno](https://www.usebruno.com/) is a fast, open-source API client. A collection for GroupScout is included in the repository.

1. **Open Bruno**.
2. **Open Collection**: Point it to the `api/bruno` folder in this repository.
3. **Select Environment**: Click the top-right environment selector and choose **Local**.
4. **Configure Token**: Edit the **Local** environment variables to match your `API_TOKEN`.
5. **Run Requests**: You can now run all pre-configured requests.

#### 8.4 Testing with OpenAPI / Swagger
An OpenAPI 3.0 specification is available at `api/swagger.yaml`.
- **Visualizing**: You can paste the content of `api/swagger.yaml` into the [Swagger Editor](https://editor.swagger.io/) or use a local Swagger UI instance.
- **Servers**: The spec defines two servers:
    - `http://localhost:8080` (Lead Generation)
    - `http://localhost:8081` (Alerting Service)

### 10. Audit Trail & Retention Testing

The audit trail includes privacy and retention features that should be verified regularly.

#### Test PII Redaction
Verify that emails and phone numbers are redacted when `PII_STRIP=true` is enabled:
```bash
# This test specifically covers PII stripping logic
go test -v ./internal/storage -run TestAuditStore_StripPII
```

#### Test Raw Audit Preview Safety
Raw audit preview is stricter than authenticated raw access. Before any inline raw-payload display ships, tests must prove:

- Default lead list, lead detail, and verification queue responses do not contain raw body fields or raw audit IDs.
- The planned `GET /api/leads/{id}/raw` alias remains authenticated and returns stored bytes only after explicit raw-route access. The current legacy `GET /leads/{id}/raw` route in the inspected backend source snapshot is not browser-safe and does not enforce the planned admin/session boundary.
- Preview rendering redacts secrets, credentials, cookies, signed URLs, database URLs, webhook URLs, emails, phones, and individual contact names.
- JSON/XML/plain text/sanitized HTML are the only inline-preview payload classes; PDFs, images, archives, office documents, unknown binary, and oversized payloads stay download/open-only.
- Browser/static asset scans still reject `API_TOKEN`, provider keys, database URLs, Slack/email secrets, session secrets, and literal bearer tokens.

#### Test Retention Purge
Verify that old records are deleted, but referenced ones are kept:
```bash
# This test verifies that PurgeOlderThan ignores referenced records
go test -v ./internal/storage -run TestAuditStore_PurgeOlderThan

# This test verifies the retention worker cutoff/result behavior
go test -v ./internal/auditretention

# This test verifies Postgres preserves referenced raw inputs during purge
TEST_POSTGRES_URL="postgres://groupscout:groupscout@localhost:5432/groupscout?sslmode=disable" \
  go test -v -tags integration ./internal/storage -run TestAuditStore_PurgeOlderThan_PostgresPreservesReferencedRawInputs
```

#### Manual Verification via SQL
You can verify the state of your live database (Postgres) using the queries provided in [POSTGRES_QUERIES.md](./POSTGRES_QUERIES.md#audit-trail-raw-inputs).
