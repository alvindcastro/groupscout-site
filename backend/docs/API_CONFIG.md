### API & Endpoint Configuration Guide

This document lists all internal and external API endpoints used by GroupScout and specifies which are configurable via environment variables.

---

### 1. Internal API Endpoints (Inbound)

These endpoints are exposed by the GroupScout binaries for external triggers (e.g., n8n, Slack, or monitoring).

#### Lead Generation Server (`cmd/server`)
Default port: `8080` (Configurable via `PORT`)

| Endpoint | Method | Description | Auth Required |
|---|---|---|---|
| `/health` | `GET` | System health check (DB + API connectivity). | No |
| `/metrics` | `GET` | Prometheus endpoint for currently wired runtime metrics; collected/enriched/notified/failure counters, collector freshness, and dashboard docs remain open in `groupscout-site-yyj`. | No |
| `/run` | `POST` | Manually triggers the collect-enrich-notify pipeline. | Yes (Bearer Token) |
| `/digest` | `POST` | Sends a weekly email summary of leads via Resend. | Yes (Bearer Token) |
| `/n8n/webhook` | `POST` | Receives a lead-shaped JSON payload from n8n, inserts it directly, and optionally notifies Slack. Raw single-item enrichment is planned separately in `groupscout-site-b25`. | Yes (Bearer Token) |
| `/leads/{id}/raw` | `GET` | Current legacy raw audit payload route. The inspected backend source snapshot does not enforce bearer or admin-session auth here; do not expose this route to browser UI until auth/sanitized preview work lands. | No in current snapshot |
| `/api/auth/status` | `GET` | Planned admin auth status endpoint; not live in the inspected backend source snapshot. | Planned |
| `/api/auth/login` | `POST` | Planned setup-token login for a `groupscout_session` cookie; not live in the inspected backend source snapshot. | Planned |
| `/api/auth/logout` | `POST` | Planned admin-session revoke endpoint; not live in the inspected backend source snapshot. | Planned |
| `/api/auth/me` | `GET` | Planned current-admin endpoint; not live in the inspected backend source snapshot. | Planned |

#### Alertd Binary (`cmd/alertd`)
Default port: `8081` (Configurable via `ALERTD_PORT`)

| Endpoint | Method | Description | Auth Required |
|---|---|---|---|
| `/slack/inventory` | `POST` | Slack slash command to update real-time room availability. | No (Slack verification recommended) |

---

### 2. External Service Integrations (Outbound)

GroupScout integrates with several external SaaS providers.

| Service | Environment Variable | Default / Base URL | Purpose |
|---|---|---|---|
| **Anthropic** | `CLAUDE_API_KEY` | `https://api.anthropic.com/v1/messages` | AI lead enrichment and outreach drafting. |
| **Google Gemini** | `GEMINI_API_KEY` | `https://generativelanguage.googleapis.com/...` | Alternative AI provider. |
| **Slack Webhooks** | `SLACK_WEBHOOK_URL` | `https://hooks.slack.com/services/...` | Posting lead digests to channels. |
| **Slack Bot API** | `SLACK_BOT_TOKEN` | `https://slack.com/api/...` | Used by `alertd` for stateful alert updates. |
| **Resend** | `RESEND_API_KEY` | `https://api.resend.com/emails` | Sending weekly lead summary emails. |
| **Hunter.io** | `HUNTER_API_KEY` | `https://api.hunter.io/v2/...` | (Phase 18) Finding decision-maker contact info. |
| **Sentry** | `SENTRY_DSN` | - | Error tracking and observability. |

---

### 3. Data Source Polling (Scrapers)

The following URLs are polled by collectors to gather raw data.

| Source | Environment Variable | Current Default URL | Status |
|---|---|---|---|
| **Richmond Permits** | `RICHMOND_PERMITS_URL` | `https://www.richmond.ca/.../*.pdf` | Configurable |
| **Delta Permits** | `DELTA_PERMITS_URL` | `https://delta.ca/building-permits` | Configurable |
| **CreativeBC** | `CREATIVEBC_URL` | `https://www.creativebc.com/...` | Configurable |
| **VCC Events** | `VCC_URL` | `https://www.vancouverconventioncentre.com/...` | Configurable |
| **BC Bid RSS** | `BCBID_RSS_URL` | `https://www.civicinfo.bc.ca/rss/...` | Configurable |
| **News RSS** | `NEWS_RSS_URL` | `https://news.google.com/rss/...` | Configurable |
| **Eventbrite** | `EVENTBRITE_URL` | `https://www.eventbrite.ca/...` | Configurable |
| **ECCC Weather** | `ECCC_ALERTS_URL` | `https://api.weather.gc.ca/collections/alerts/items` | Configurable |
| **YVR Flight Status**| `YVR_FLIGHT_STATUS_URL` | `https://www.yvr.ca/en/passengers/flights` | Configurable |
| **NavCanada NOTAM** | `NAVCANADA_NOTAM_URL` | `https://www.navcanada.ca/.../notam-portal.aspx` | Configurable |

---

### 4. Implementation Notes

-   **Internal APIs**: The Lead Generation Server and Alertd are separate binaries with distinct ports.
-   **Security**: Use the `API_TOKEN` (Bearer Auth) for all sensitive POST endpoints.
-   **Extensibility**: Adding new data sources usually requires a new entry in the Scrapers table and a corresponding URL in the `.env` file.
-   **Testing**: See [TESTING.md](./guides/TESTING.md#8-api-testing-details) for a guide on how to test these endpoints with curl, Bruno, or Swagger.

---

### 5. UI-Facing API Namespace

The automation API remains optimized for n8n and server-to-server triggers. The operator UI uses a separate `/api/*` namespace so browser sessions, authorization, pagination, and response shapes can evolve without breaking automation clients.

Do **not** expose `API_TOKEN` to browser JavaScript. It is intended for server-to-server automation. Use a cookie session, an auth proxy, or another server-side session boundary for the admin UI.

Status reconciliation, 2026-05-25: the current backend source snapshot does not expose the planned `/api/*` operator UI routes below. Treat them as planned contracts until `groupscout-site-eqm` updates the backend source and OpenAPI, then `groupscout-site-29q` regenerates frontend client/types. The current Docker UI smoke may therefore see `/api/system` return `404` from backend `main`.

### Admin Auth And Setup Token Flow

Status reconciliation, 2026-05-25: admin auth/session is a planned UI contract under `groupscout-site-eqm`; the inspected backend source snapshot has no `ADMIN_AUTH_*` config, setup-token file handling, `groupscout_session`, or `/api/auth/*` routes.

The planned admin auth flow defaults to enabled through `ADMIN_AUTH_ENABLED=true`. If `ADMIN_SETUP_TOKEN` is absent, the server should read or create `ADMIN_SETUP_TOKEN_FILE`, default `data/admin-setup-token`. Use that setup token at `/admin/login` or `POST /api/auth/login` after the backend implementation lands.

Successful login creates a short-lived in-memory admin session, sets the HttpOnly `groupscout_session` cookie, and returns a `session_token` for non-browser smoke clients. File-backed setup tokens rotate after successful login; read `data/admin-setup-token` again before another setup login. Env-backed `ADMIN_SETUP_TOKEN` values cannot rotate automatically, so rotate them by changing the env var and restarting the backend.

Session lifetime is controlled by `ADMIN_SESSION_TTL_HOURS`, default `24`. `POST /api/auth/logout` revokes the current session and expires `groupscout_session`. Backend restarts clear in-memory sessions.

| Endpoint | Method | Status | Purpose |
|---|---|---|---|
| `/api/auth/status` | `GET` | Planned | Public auth status endpoint used by the browser before rendering protected routes. |
| `/api/auth/login` | `POST` | Planned | Setup-token login that creates a short-lived admin session and rotates file-backed setup tokens. |
| `/api/auth/logout` | `POST` | Planned | Revokes the current session and clears `groupscout_session`. |
| `/api/auth/me` | `GET` | Planned | Session-backed current-admin lookup. |
| `/api/leads` | `GET` | Planned | List leads with filters such as `status`, `source`, `min_score`, `q`, `limit`, and `cursor`. |
| `/api/leads/{id}` | `GET` | Planned | Return lead detail, enrichment fields, status, notes, and audit metadata without raw payload bodies. |
| `/api/leads/{id}` | `PATCH` | Planned | Update safe fields and apply validated actions: claim, dismiss, snooze, flag, contacted, won, lost, no-response, and reopen. |
| `/api/leads/{id}/raw` | `GET` | Planned | Authenticated UI alias for raw audit evidence. Requires admin session when admin auth is enabled, or bearer `API_TOKEN` when configured. |
| `/api/leads/{id}/outreach` | `GET` | Planned | List outreach attempts and outcomes for the lead. |
| `/api/leads/{id}/outreach` | `POST` | Planned | Log an outreach attempt, channel, contact, notes, and outcome. |
| `/api/pipeline/runs` | `POST` | Planned | Start a pipeline run asynchronously instead of blocking the browser for the full run. |
| `/api/pipeline/runs` | `GET` | Planned | Show recent run history, status, counts, and failures. |
| `/api/stats` | `GET` | Planned | Summaries by status, source, score band, owner, week, and outcome. |
| `/api/system` | `GET` | Planned | UI-friendly health and integration summary without browser-side Prometheus parsing. |
| `/api/alerts` | `GET` | Planned | Read-only alert-console compatibility endpoint. Returns an empty collection until `alertd` has a shared persistent alert store. |

See [UI_PHASE35_API_CONTRACT.md](./planning/ui/UI_PHASE35_API_CONTRACT.md) for the planned lead API response shapes and [UI_STRATEGY.md](./planning/ui/UI_STRATEGY.md) for the product flow and implementation sequence.
