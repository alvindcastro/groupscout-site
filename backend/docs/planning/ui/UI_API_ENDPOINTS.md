# UI API Endpoint Brainstorm

> Brainstormed API contracts for the future GroupScout operator UI.
> This is not an implementation record. Production code must be built through the strict TDD prompt pack in `../../prompts/PROMPTS_PHASE31_UI.md`.

## Current Backend Endpoints

| Endpoint | Status | UI note |
|---|---|---|
| `GET /health` | Implemented | Returns JSON with `status`, `database`, and `ollama`. Swagger currently describes a plain-text response, so contract docs need correction before generated clients rely on it. |
| `GET /metrics` | Implemented | Prometheus output. Useful for observability, not a direct UI data source. |
| `POST /run` | Implemented | Bearer-token protected when `API_TOKEN` is set. Blocking pipeline trigger. |
| `POST /digest?to=` | Implemented | Bearer-token protected when `API_TOKEN` is set. Sends email digest. |
| `POST /n8n/webhook` | Implemented | Bearer-token protected when `API_TOKEN` is set. Accepts a lead-shaped payload. |
| `GET /leads/{id}/raw` | Implemented | Legacy raw audit payload lookup. Now requires `API_TOKEN` or admin session when either auth mode is configured. |
| `GET /api/auth/status` | Implemented | Public admin auth status and session check. |
| `POST /api/auth/login` | Implemented | Setup-token login. File-backed setup tokens rotate after successful login. |
| `POST /api/auth/logout` | Implemented | Session revocation and cookie clearing. |
| `GET /api/auth/me` | Implemented | Current admin lookup for valid session cookie or bearer session token. |
| `GET /api/leads` | Implemented Phase 35 | Filtered lead inbox list with `status`, `source`, `min_score`, `q`, `limit`, and `cursor`. |
| `GET /api/leads/{id}` | Implemented Phase 35 | Lead detail with safe audit metadata and no raw payload body. |
| `PATCH /api/leads/{id}` | Implemented Phase 35 | Updates `status` and `notes`; rejects unsafe or not-yet-schema-backed fields. |
| `GET /api/leads/{id}/raw` | Implemented Phase 35 | Raw audit evidence alias requiring admin session when admin auth is enabled, or bearer auth when `API_TOKEN` is configured. |
| `GET /api/leads/{id}/outreach` | Implemented Phase 36 | Lists outreach history newest-first. |
| `POST /api/leads/{id}/outreach` | Implemented Phase 36 | Logs outreach attempts. |
| `POST /api/pipeline/runs` | Implemented Phase 37 | Starts a persisted asynchronous pipeline run. |
| `GET /api/pipeline/runs` | Implemented Phase 37 | Lists persisted run history with status, counts, and errors. |
| `GET /api/stats` | Implemented Phase 37 | Returns status, source, score band, owner, week, and outreach outcome summaries. |
| `GET /api/system` | Implemented Phase 37 | Returns UI-safe system health without browser-side `/metrics` parsing. |
| `GET /api/alerts` | Implemented compatibility route | Read-only alert-console endpoint; returns an empty collection until `alertd` writes alert state to shared storage. |
| `POST /slack/inventory` | Implemented in `alertd` | Slack slash-command endpoint, separate from the lead UI MVP. |

## Contract Principles

- Browser-facing routes should live under same-origin `/api/*`.
- Browser code must not receive `API_TOKEN`.
- API responses should be JSON except raw audit payload downloads or previews.
- Pagination should use cursors before offsets if the lead list will grow.
- Field corrections must preserve original extracted values and reviewer edits. Do not silently overwrite source-backed extraction.
- Commercial workflow state and verification state may need to be separate fields.

## Proposed Endpoints

| Endpoint | Purpose | Request | Response |
|---|---|---|---|
| `GET /api/leads` | Lead inbox | `status`, `source`, `min_score`, `q`, `limit`, `cursor` query params | `{items:[lead_summary], next_cursor, filters}` |
| `GET /api/leads/{id}` | Lead detail | path ID | `{lead, audit, outreach_summary, activity}` |
| `PATCH /api/leads/{id}` | Safe lead update | `{status?, notes?}` | `{lead, changed_fields, updated_at}` |
| `GET /api/leads/{id}/raw` | Authenticated raw evidence alias | path ID | raw bytes with stored content type |
| `GET /api/leads/{id}/outreach` | Outreach history | path ID, `limit`, `cursor` | `{items:[outreach_event], next_cursor}` |
| `POST /api/leads/{id}/outreach` | Log outreach attempt | `{contact, channel, notes, outcome}` | `{outreach, lead}` |
| `POST /api/pipeline/runs` | Browser-safe run trigger | `{sources?, bcbid_raw_input?, dry_run?}` | `{run_id, status, started_at}` |
| `GET /api/pipeline/runs` | Run history | `status`, `limit`, `cursor` | `{items:[pipeline_run], next_cursor, filters}` |
| `GET /api/stats` | UI summaries | `window` | `{window, by_status, by_source, score_bands, by_owner, by_week, by_outcome}` |
| `GET /api/system` | UI health summary | none | `{status, database, ollama, metrics_available, last_pipeline_run}` |
| `GET /api/alerts` | Read-only alert console compatibility | `state`, `property`, `limit`, `cursor` | `{alerts:[], items:[], next_cursor, read_only, filters}` |
| `GET /api/auth/status` | Admin session status | none | `{auth_required, authenticated, setup_token_file}` |
| `POST /api/auth/login` | Admin setup-token login | `{token}` | `{session_token, token_type, expires_at, setup_token_rotated, user}` plus `groupscout_session` cookie |
| `POST /api/auth/logout` | Admin logout | cookie or bearer session token | `{revoked}` plus expired `groupscout_session` cookie |
| `GET /api/auth/me` | Current admin | cookie or bearer session token | `{user:{role:"admin"}}` |

Admin setup-token notes:

- `ADMIN_AUTH_ENABLED` defaults to `true`.
- If `ADMIN_SETUP_TOKEN` is absent, the backend reads or creates `ADMIN_SETUP_TOKEN_FILE`, default `data/admin-setup-token`.
- File-backed setup tokens rotate after successful login; env-backed `ADMIN_SETUP_TOKEN` values do not.
- Admin sessions are in-memory and expire after `ADMIN_SESSION_TTL_HOURS`, default `24`.
- Browser clients use the HttpOnly `groupscout_session` cookie. Non-browser smoke clients may use the returned `session_token` as `Authorization: Bearer ...`.

## Response Shape Notes

`lead_summary` should include:

- `id`
- `title`
- `source`
- `location`
- `project_value`
- `priority_score`
- `priority_reason`
- `status`
- `created_at`
- `updated_at`
- `has_raw`
- `audit_source_url`

Historical note: Phase 35 implemented `lead_summary` without `owner` or `verification_state`; Phase 36 schema-backed those fields. `groupscout-site-dxq` tracks reconciling any remaining stale Phase 35-era schema notes.

`lead` should include full current storage fields plus UI-safe audit metadata. Do not include raw payload bodies by default. Phase 34 detail fixtures should mirror this as `{lead, audit, outreach_summary, activity}` where `audit` contains metadata such as `has_raw`, `raw_link`, `payload_type`, `source_url`, `collector_name`, and `collected_at`, never the raw `payload` body.

`outreach_event` should include:

- `id`
- `lead_id`
- `contact`
- `channel`
- `notes`
- `outcome`
- `logged_at`

`pipeline_run` should include:

- `id`
- `status`
- `started_at`
- `finished_at`
- `sources`
- `counts`
- `errors`

`alert` should include the frontend Phase 10 read-only fields when a shared alert store exists:

- `id`
- `property`
- `sps`
- `state`
- `impact`
- `updated_at`
- `evidence`
- `room_inventory`
- `action_history`

The current backend compatibility route returns empty `alerts` and `items` arrays because `alertd` still keeps runtime alert state outside the lead API database. Tracked follow-up: `groupscout-site-3gq`.

## Schema Gaps To Handle Explicitly

The current database schema supports core lead fields, status, notes, raw input IDs, owner/snooze/flag workflow fields, `verification_state`, `outreach_log`, and persisted `pipeline_runs`. Some older Phase 35 notes predate Phase 36/37; `groupscout-site-dxq` tracks reconciling any remaining stale roadmap/schema notes.

| Planned UI field | Current status | Planning action |
|---|---|---|
| `owner` | Implemented Phase 36 | Keep list/detail/UI docs aligned with live API responses. |
| `snoozed_until` | Implemented Phase 36 | Keep list/detail/UI docs aligned with live API responses. |
| `verification_state` | Implemented Phase 36 | Stored separately from commercial workflow status. |
| `corrections` | Missing | Needs audit-safe correction model before UI edit controls are live. |
| `pipeline_runs` | Implemented Phase 37 | Keep smoke/docs aligned with persisted run history. |
