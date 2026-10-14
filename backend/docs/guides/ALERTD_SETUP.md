# alertd — Airport Disruption Alert System

`alertd` is a specialized binary within the GroupScout ecosystem that monitors airport disruptions (weather, cancellations, ground stops) and alerts hotel teams via Slack.

## Architecture

- **Polling Loop:** Runs in the background with two modes:
  - **Quiet Mode (Default):** Polls every 10 minutes.
  - **Active Mode:** Polls every 90 seconds when an alert is active or growing.
- **SPS (Stranded Passenger Score):** A proprietary formula that quantifies the impact on hotel demand.
- **State Machine:** Manages the lifecycle of an event (`Watch → Alert → Updating → Resolved`).
- **Slack Integration:** Uses a Slack Bot token to post and update threaded messages.

## Configuration

Hotel configurations are stored in `config/airports.yaml`.

```yaml
- id: sandman-richmond
  name: Sandman Richmond
  slack_channel: "#alerts-yvr"
  airports:
    - code: CYVR
      distance_min: 8
  alert_threshold_sps: 60
  distressed_rate: 119
  rack_rate: 220
  airline_contacts:
    - carrier: Air Canada
      ops_phone: "604-276-7477"
```

### Environment Variables

- `SLACK_BOT_TOKEN`: (Required) The Bot User OAuth Token for the Slack app.
- `ALERTD_CONFIG_PATH`: (Optional) Custom path to the YAML config (defaults to `config/airports.yaml`).
- `ALERTD_PORT`: (Optional) Port for the Slack slash command HTTP server (defaults to `8081`).
- `OLLAMA_ENABLED`, `OLLAMA_ALERT_COPY_ENABLED`, `OLLAMA_ENDPOINT`, `OLLAMA_ALERT_COPY_TIMEOUT_S`: (Optional) Ollama-generated alert copy. It is used only when both flags are `true`, prefers `llama3.1:8b`, and falls back to `mistral`.

`alertd` reads nothing else. In Docker Compose it receives only these variables: `SLACK_BOT_TOKEN` and the `OLLAMA_*` flags from `.env`, `ALERTD_PORT=8081`, and `OLLAMA_ENDPOINT=http://ollama:11434`. It does not get the whole `.env` (no `env_file`), so pipeline secrets such as `CLAUDE_API_KEY` never reach it. This applies from IaC Phase 1, backend branch `task/iac-phase1-hardening`.

## Slack Slash Commands

`alertd` includes an HTTP server to handle Slack slash commands.

### `/inventory N`

Updates the current room inventory for all alerts.
- **Usage:** `/inventory 34`
- **Effect:** Subsequent Slack alerts will display "34 rooms available".
- **Setup:**
  1. Go to your Slack App settings.
  2. Navigate to **Slash Commands**.
  3. Create a new command `/inventory`.
  4. Request URL: `http://your-server:8081/slack/inventory`.

If room inventory is not set, alerts will display: "room count not set — use /inventory N".

## Deployment

`alertd` runs as a separate core service in `docker-compose.yml`. It uses the same image as `groupscout`, with `command: ["./alertd"]`:

```bash
docker compose up -d alertd
```

Without `SLACK_BOT_TOKEN`, `alertd` exits at startup (`SLACK_BOT_TOKEN is required`), and `restart: unless-stopped` keeps restarting it.

### Health

`GET /health` on port `8081` returns `{"status":"ok","version":"<git sha>"}` while the HTTP server is up. The Compose healthcheck uses it, so `docker compose ps` shows `alertd` as `(healthy)`.

## Monitoring & Logs

`alertd` logs to stdout in the same format as the main GroupScout server. It includes exponential backoff logs if external APIs (ECCC, YVR, NavCanada) fail.
