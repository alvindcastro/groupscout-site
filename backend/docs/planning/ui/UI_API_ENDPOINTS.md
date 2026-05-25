# UI API Endpoint Brainstorm

> Brainstormed API contracts for the future GroupScout operator UI.
> This is not an implementation record. Production code must be built through the strict TDD prompt pack in `../../prompts/PROMPTS_PHASE31_UI.md`.

## Current Backend Snapshot

Status reconciliation, 2026-05-25: backend `main` currently exposes the automation/server routes listed below and the legacy raw-audit route. The UI-specific `/api/*` lead, outreach, pipeline, stats, system, auth, and alert routes are product/API contracts in this planning pack, not live routes on backend `main`. The Docker UI smoke intentionally accepts `/api/system` returning `404` from backend `main`; generated frontend clients must not treat these `/api/*` contracts as live until `groupscout-site-eqm` implements or merges the backend API surface and `groupscout-site-29q` regenerates frontend client/types.

| Endpoint | Status | UI note |
|---|---|---|
| `GET /health` | Implemented | Returns JSON with `status`, `database`, and `ollama`. Swagger currently describes a plain-text response, so contract docs need correction before generated clients rely on it. |
| `GET /metrics` | Implemented | Prometheus output. Useful for observability, not a direct UI data source. |
| `POST /run` | Implemented | Bearer-token protected when `API_TOKEN` is set. Blocking pipeline trigger. |
| `POST /digest?to=` | Implemented | Bearer-token protected when `API_TOKEN` is set. Sends email digest. |
| `POST /ingest` | Implemented in backend branch `task/event-driven-ingest` | Bearer-token protected when `API_TOKEN` is set. Accepts one raw project/event payload and runs `EnrichOne()` through the dedup, raw audit, scoring, enrichment, and lead storage path. Not a browser UI route. |
| `POST /n8n/webhook` | Implemented | Bearer-token protected when `API_TOKEN` is set. Accepts a pre-enriched lead-shaped payload and direct-inserts it. |
| `GET /leads/{id}/raw` | Implemented legacy route | Raw audit payload lookup. The inspected backend source snapshot does not enforce bearer or admin-session auth here; do not expose it directly to browser UI. Sanitized/authenticated preview remains tracked by `groupscout-site-4cv`. |
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

Historical note: Phase 35 planned `lead_summary` without `owner` or `verification_state`; Phase 36 later planned schema-backed workflow fields. The current backend source snapshot does not yet expose these fields through `/api/leads`.

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

Planned/branch-history compatibility returns empty `alerts` and `items` arrays until `alertd` has a shared persistent alert store. The inspected backend source snapshot does not currently expose `/api/alerts`. Tracked follow-up: `groupscout-site-3gq`.

## Live Schema And API Gaps To Handle Explicitly

The current backend source snapshot supports core lead fields, status, notes, raw input IDs, and the legacy `outreach_log` table. Migration files exist for owner/snooze/flag workflow fields and `pipeline_runs`, but the current `Lead` model, lead store methods, `/api/*` handlers, stats queries, auth/session routes, and OpenAPI source are not live on backend `main`.

| Planned UI field | Current status | Planning action |
|---|---|---|
| `owner` | Migration planned/present; not exposed by current lead model/API | Implement with lead store, action validation, tests, and OpenAPI before UI relies on it. |
| `snoozed_until` | Migration planned/present; not exposed by current lead model/API | Implement with lead store, action validation, tests, and OpenAPI before UI relies on it. |
| `flagged` | Migration planned/present; not exposed by current lead model/API | Implement with lead store, action validation, tests, and OpenAPI before UI relies on it. |
| `verification_state` | Migration planned/present; not exposed by current lead model/API | Keep separate from commercial workflow status when implementation lands. |
| `corrections` | Missing | Needs audit-safe correction model before UI edit controls are live. |
| `pipeline_runs` | Migration planned/present; no current store/API route | Implement persisted run store, async trigger, stats/system routes, tests, and OpenAPI before UI relies on it. |
