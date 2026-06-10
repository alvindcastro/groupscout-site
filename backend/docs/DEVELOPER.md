# Developer Guide — GroupScout

This guide provides technical details for developers working on the `groupscout` project, including the main lead generation server and the `alertd` airport disruption monitor.

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/backend/docs/DEVELOPER.md`. Run backend implementation, build, test, and Docker commands from the backend source repo at `/mnt/c/Users/alvin/GolandProjects/groupscout`. Long-lived backend and frontend markdown docs now live in `/mnt/c/Users/alvin/groupscout-site`.

## 🛠 Project Management (Makefile)

The `Makefile` is the central hub for common development tasks. Run `make help` to see all available commands.

### Common Tasks:
- `make build`: Build `server` and `alertd` binaries.
- `make test`: Run all Go tests.
- `make run`: Run the lead generation server.
- `make docker-up`: Start the full Docker stack.
- `make clean`: Remove build artifacts.

### Backend Plus UI Docker Smoke

The separate UI repo lives at `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

Use [BACKEND_FRONTEND_DOCKER_E2E.md](./planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md) when you want the backend Compose stack and the UI Docker image running together. The current flow starts backend service `groupscout`, validates the UI D3 health harness on port `3001`, then runs the UI D4 production static/proxy container on port `3002` attached to `groupscout_groupscout_net`.

Important distinction: UI port `3001` is the Phase 13 product dev server for generated app assets and `/healthz`. The production static/proxy smoke is the D4 container on `3002`.

Use [BACKEND_FOR_UI_TESTING.md](./planning/ui/BACKEND_FOR_UI_TESTING.md) for local backend startup notes and [TESTING.md](./guides/TESTING.md) for test commands. Backend health may return `200` with database OK while reporting Ollama unavailable; treat that as a blocker only for LLM/enrichment testing.

### Local n8n Developer Checks

The Compose n8n UI is at `http://localhost:5678`. For the Sunday/Wednesday cadence, developers should verify both the CLI export and the visible workflow graph before handing off:

```bash
docker exec groupscout_n8n wget -qO- http://groupscout:8080/health
docker exec groupscout_n8n n8n export:workflow --id=groupscout-sunday-wednesday-lead-cadence --output=/tmp/groupscout-cadence-review.json
docker exec groupscout_n8n sh -lc "grep -o '\"active\":[^,}]*\|\"triggerAtDay\":\[[^]]*\]\|\"triggerAtHour\":[0-9]*\|\"triggerAtMinute\":[0-9]*\|\"timezone\":\"[^\"]*\"' /tmp/groupscout-cadence-review.json"
docker exec groupscout_n8n sh -lc "grep -o '\"jsonBody\":\"={{ JSON.stringify({}) }}\"\\|notified_leads\\|new_leads' /tmp/groupscout-cadence-review.json"
```

If local n8n login is unavailable, follow [Recover local n8n sign-in](./guides/TROUBLESHOOTING.md#6-recover-local-n8n-sign-in) instead of deleting the `n8n_data` volume; deleting the volume also deletes imported workflows and credentials.

## 🏗 Project Architecture

GroupScout consists of two primary Go binaries:
1.  **Lead Generation Server (`cmd/server`)**: Scrapes building permits, film productions, and events to generate hotel group sales leads.
2.  **Airport Disruption Alert System (`alertd` / `cmd/alertd`)**: Monitors YVR flight disruptions, ECCC weather alerts, and NavCanada NOTAMs to alert hotel teams via Slack.

## 🚀 Getting Started

### Prerequisites
- **Go 1.26+**
- **pdftotext** (included with Git for Windows, or install via Poppler/XPDF on Linux/macOS)
- **Docker & Docker Compose** (optional, for Postgres/n8n/monitoring)

### Environment Variables
Create a `.env` file in the project root. Essential variables:
```env
CLAUDE_API_KEY=sk-ant-...        # Anthropic API key for AI enrichment
SLACK_BOT_TOKEN=xoxb-...          # Slack Bot Token (required for alertd)
SLACK_WEBHOOK_URL=https://...     # Slack Incoming Webhook (for lead digests)
DATABASE_URL=groupscout.db        # SQLite or Postgres connection string
ALERTD_PORT=8081                  # Port for alertd slash commands (default: 8081)
```

## 🏃 Running the Binaries

### 1. Lead Generation Server
**Server Mode (API triggered):**
```bash
go run cmd/server/main.go
```
**CLI Mode (One-time run):**
```bash
go run cmd/server/main.go --run-once
```

### 2. Alertd (Disruption Monitor)
**Run Alertd:**
```bash
go run cmd/alertd/main.go
```
Alertd requires a configuration file at `config/airports.yaml` to define hotels and their monitored airports.

### 3. Ollama Modelfile Management
GroupScout uses local Ollama models for enrichment and alert copy through the implemented `internal/ollama` runtime. Modelfiles define the personas and instructions for these models. The future Phase 16 `LLM_PROVIDER=ollama` provider-abstraction path is tracked separately in planning docs.

**Push Modelfiles:**
To update or create the personas on your local Ollama server without restarting the main server:
```bash
go run cmd/server/main.go ollama push-models
```
This reads all `.modelfile` files from `internal/ollama/modelfile/` and pushes them to the Ollama endpoint.

**List Models:**
To see which models are currently loaded in your Ollama instance:
```bash
go run cmd/server/main.go ollama list-models
```

**Workflow for Updating Persona Prompts:**
1. Edit the relevant `.modelfile` in `internal/ollama/modelfile/`.
2. Run `go run cmd/server/main.go ollama push-models`.
3. The new persona is immediately available for the next LLM call.

## 🛠 New Features & How to Run

### `/inventory` Slack Slash Command (alertd)
`alertd` now includes an HTTP server that listens for Slack slash commands to update real-time room availability in alerts.

#### Local Development & Testing
To test Slack slash commands locally without a public URL:
1.  **Expose your local port** (e.g., using `ngrok`):
    ```bash
    ngrok http 8081
    ```
2.  **Configure Slack App**: Set the Request URL for the `/inventory` command to `https://<your-ngrok-url>/slack/inventory`.
3.  **Simulate with curl**:
    ```bash
    curl -X POST -d "command=/inventory&text=34" http://localhost:8081/slack/inventory
    ```

#### Verification
When `/inventory 34` is called, the current `alertd` instance updates its in-memory inventory. Subsequent Slack alerts will display "34 rooms available" instead of the "room count not set" fallback.

## 🧪 Testing

We use a combination of unit tests, integration tests, and manual verification scripts.

### 1. Automated Tests
```bash
make test
```
See [TESTING.md](./guides/TESTING.md) for details on running specific tests and Postgres integration.

Planned admin/operator login uses a setup-token flow, not `API_TOKEN`. The inspected backend source snapshot does not currently expose the `/api/auth/*` routes; see [ADMIN_AUTH.md](./guides/ADMIN_AUTH.md) for the target `ADMIN_AUTH_ENABLED`, `ADMIN_SETUP_TOKEN`, setup-token rotation, `groupscout_session`, logout, and Docker smoke details.

### 2. Ollama & LLM Testing
A dedicated script verifies Ollama connectivity and model availability:
```bash
./scripts/test-ollama.sh
```

### 3. API Testing
Manual API testing can be done via `curl` or the **Bruno** collection in `api/bruno`.
See [TESTING.md](./guides/TESTING.md#8-api-testing-details) for examples.

### 4. n8n Webhook Score Contract
`POST /n8n/webhook` is a direct-insert path for pre-enriched lead-shaped payloads. It does not run the normal collector/enrichment scorer, so boundary normalization happens in the webhook handler before storage and before Slack notification.

Use `PriorityScore` on the Slack/internal `0-10` scale. Legacy workflows that still send percent-style `0-100` values are accepted and normalized (`90 -> 9`, `98 -> 10`) so Slack and email never show impossible scores such as `98/10`. Defensive display clamping also protects older bad rows already in storage.

## 📂 Project Structure
- `api/`: OpenAPI / Swagger specifications.
- `cmd/`: Entry points for `server`, `alertd`, and dev tools.
- `config/`: Centralized environment and YAML configuration.
- `.claude/agents/`: Source-of-truth definitions for specialized backend agents in the backend source repo.
- `backend/plugins/groupscout-agents/`: Repo-local Codex skill bundle derived from the `.claude/agents/` specs.
- `docs/`: Historical source location for in-depth guides and planning documents. Long-lived markdown docs are now centralized under `/mnt/c/Users/alvin/groupscout-site/backend`.
- `internal/`: Core business logic (scrapers, scoring, state machines, storage).
- `migrations/`: SQL migration files (for Postgres/SQLite).

## 🤖 Specialized Agents

This project uses specialized backend agent roles to streamline development in different domains. Role boundaries are maintained in the backend source repo's `.claude/agents/` specs; the coordinator repo mirrors them as Codex skills under `backend/plugins/groupscout-agents/` for Codex workflows.

See [SUBAGENTS.md](./guides/SUBAGENTS.md) for a list of available agents and how to use them.

## 📄 Related Documentation
- [README.md](../README.md) - Project overview and user setup.
- [ARCHITECTURE.md](./ARCHITECTURE.md) - System design and data flow.
- [CHANGELOG.md](./CHANGELOG.md) - Plain-English record of all changes.
- [DOCKER.md](./guides/DOCKER.md) - Running and troubleshooting Docker.
- [ADMIN_AUTH.md](./guides/ADMIN_AUTH.md) - Admin setup-token login, session cookie, logout, and token rotation.
- [BACKEND_FRONTEND_DOCKER_E2E.md](./planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md) - Current backend plus separate UI Docker smoke path.
- [SUBAGENTS.md](./guides/SUBAGENTS.md) - How to use specialized backend agents.
- [TROUBLESHOOTING.md](./guides/TROUBLESHOOTING.md) - Pipeline and missing lead troubleshooting.
- [SETUP.md](./guides/SETUP.md) - Detailed environment and dependency setup.
- [TESTING.md](./guides/TESTING.md) - Comprehensive testing guide.
- [N8N_GUIDE.md](./guides/N8N_GUIDE.md) - Workflow automation and scheduling.
- [HOME_DEPLOY.md](./guides/HOME_DEPLOY.md) - Self-hosting and deployment guide.
- [API_CONFIG.md](./API_CONFIG.md) - Detailed API and endpoint configuration.
- [TESTING API examples](./guides/TESTING.md#8-api-testing-details) - Guide on how to test the APIs.
- [ALERTD_SETUP.md](./guides/ALERTD_SETUP.md) - Specific configuration for the alert system.
- [OLLAMA_INTEGRATION.md](./planning/OLLAMA_INTEGRATION.md) - Local LLM integration plan and phases.
- [OLLAMA_SETUP.md](./guides/OLLAMA_SETUP.md) - Docker and native setup guide for Ollama.
- [PHASES.md](./planning/PHASES.md) - Build tracker and phase history.
- [ROADMAP.md](./planning/ROADMAP.md) - Long-term project roadmap.
- [AUDIT_TRAIL.md](./planning/AUDIT_TRAIL.md) - Input storage and verification plan.
- [PROMPTS.md](./planning/PROMPTS.md) - Prompt library and Strict TDD guide.
