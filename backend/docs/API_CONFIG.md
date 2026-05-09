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
| `/metrics` | `GET` | Prometheus metrics for runtime monitoring. | No |
| `/run` | `POST` | Manually triggers the collect-enrich-notify pipeline. | Yes (Bearer Token) |
| `/digest` | `POST` | Sends a weekly email summary of leads via Resend. | Yes (Bearer Token) |
| `/n8n/webhook` | `POST` | Receives raw lead data from n8n for storage and enrichment. | Yes (Bearer Token) |
| `/leads/{id}/raw` | `GET` | Legacy raw audit payload route. | Yes when `API_TOKEN` or admin auth is configured |
| `/api/auth/status` | `GET` | Reports admin auth requirement and current session status. | No |
| `/api/auth/login` | `POST` | Exchanges the current admin setup token for a `groupscout_session` cookie. File-backed setup tokens rotate after successful login. | Setup token |
| `/api/auth/logout` | `POST` | Revokes the current admin session and clears the browser cookie. | No; revokes when a session is present |
| `/api/auth/me` | `GET` | Returns the current admin user for a valid session cookie or bearer session token. | Yes (admin session) |

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

### Admin Auth And Setup Token Flow

Admin auth defaults to enabled through `ADMIN_AUTH_ENABLED=true`. If `ADMIN_SETUP_TOKEN` is absent, the server reads or creates `ADMIN_SETUP_TOKEN_FILE`, default `data/admin-setup-token`. Use that setup token at `/admin/login` or `POST /api/auth/login`.

Successful login creates a short-lived in-memory admin session, sets the HttpOnly `groupscout_session` cookie, and returns a `session_token` for non-browser smoke clients. File-backed setup tokens rotate after successful login; read `data/admin-setup-token` again before another setup login. Env-backed `ADMIN_SETUP_TOKEN` values cannot rotate automatically, so rotate them by changing the env var and restarting the backend.

Session lifetime is controlled by `ADMIN_SESSION_TTL_HOURS`, default `24`. `POST /api/auth/logout` revokes the current session and expires `groupscout_session`. Backend restarts clear in-memory sessions.

| Endpoint | Method | Status | Purpose |
|---|---|---|---|
| `/api/auth/status` | `GET` | Implemented | Public auth status endpoint used by the browser before rendering protected routes. |
| `/api/auth/login` | `POST` | Implemented | Setup-token login that creates a short-lived admin session and rotates file-backed setup tokens. |
| `/api/auth/logout` | `POST` | Implemented | Revokes the current session and clears `groupscout_session`. |
| `/api/auth/me` | `GET` | Implemented | Session-backed current-admin lookup. |
| `/api/leads` | `GET` | Implemented | List leads with filters such as `status`, `source`, `min_score`, `q`, `limit`, and `cursor`. |
| `/api/leads/{id}` | `GET` | Implemented | Return lead detail, enrichment fields, status, notes, and audit metadata without raw payload bodies. |
| `/api/leads/{id}` | `PATCH` | Implemented | Update safe fields and apply validated actions: claim, dismiss, snooze, flag, contacted, won, lost, no-response, and reopen. |
| `/api/leads/{id}/raw` | `GET` | Implemented | Authenticated UI alias for raw audit evidence. Requires admin session when admin auth is enabled, or bearer `API_TOKEN` when configured. |
| `/api/leads/{id}/outreach` | `GET` | Implemented | List outreach attempts and outcomes for the lead. |
| `/api/leads/{id}/outreach` | `POST` | Implemented | Log an outreach attempt, channel, contact, notes, and outcome. |
| `/api/pipeline/runs` | `POST` | Implemented | Start a pipeline run asynchronously instead of blocking the browser for the full run. |
| `/api/pipeline/runs` | `GET` | Implemented | Show recent run history, status, counts, and failures. |
| `/api/stats` | `GET` | Implemented | Summaries by status, source, score band, owner, week, and outcome. |
| `/api/system` | `GET` | Implemented | UI-friendly health and integration summary without browser-side Prometheus parsing. |
| `/api/alerts` | `GET` | Implemented | Read-only alert-console compatibility endpoint. Returns an empty collection until `alertd` has a shared persistent alert store. |

See [UI_PHASE35_API_CONTRACT.md](./planning/ui/UI_PHASE35_API_CONTRACT.md) for the implemented lead API response shapes and [UI_STRATEGY.md](./planning/ui/UI_STRATEGY.md) for the product flow and implementation sequence.
