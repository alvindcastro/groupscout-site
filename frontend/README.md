# GroupScout UI

Phase 0-15 now follows the canonical `UI_TDD_PHASE_PROMPTS.md` order: baseline reconciliation, product IA/UX guardrails, backend compatibility smoke, API client contracts, Today, Leads, Lead Detail, status/corrections, verification, outreach, pipeline, analytics, read-only alerts, session/runtime, Docker E2E, and browser UX hardening. The restored baseline includes the dependency-free vanilla DOM product renderer/runtime, static build, product dev server, production static/proxy server, and same-origin `/api/*` contracts.

## Current Scope

- Design tokens live in `web/src/design/tokens.js` and are mapped from `DESIGN.md`.
- The route shell lives in `web/src/app/shell.js` and mounts Today, Leads, Verification, Outreach, Pipeline, Analytics, Alerts, and a Settings placeholder.
- Browser API access still enters through `web/src/api/client.js`, with same-origin `/api/*` transport centralized in `web/src/api/transport.js` and feature adapters split across focused `web/src/api/*` modules.
- UI deployment/session rules live in `web/src/server/uiDeployment.js` and cover `UI_ENABLED`, `UI_BASE_PATH`, `UI_SESSION_SECRET`, development-only `CORS_ALLOWED_ORIGINS`, session-cookie `/api/*` access when configured, and a no-secret Docker smoke mode.
- Browser runtime contract metadata lives in `web/src/server/browserRuntimeContract.js`; D4 now implements the lightweight production Node server on port `3000` with `/healthz` and same-origin `/api/*` routing to server-side `http://groupscout:8080` without exposing automation credentials.
- Phase 13 renderer/runtime metadata lives in `web/src/server/productRendererRuntime.js`; the first rendered-route smoke coverage lives in `test/phase-13-renderer-runtime.test.js`.
- Phase 15 browser UX hardening metadata lives in `web/src/renderer/browserUxHardening.js`; deterministic route-specific focus, accessible-name, rendered responsive-mode, state-region, text-containment, and same-origin checks live in `test/browser-ux-hardening.test.js`.
- Production same-origin serving lives in `web/src/server/productionServer.js`; `npm run start:ui` serves `web/dist`, falls back to `index.html` for app routes, applies browser security headers, gates `/api/*` with `groupscout_session` when `UI_SESSION_SECRET` is configured, and otherwise forwards Docker-smoke API requests server-side to `UI_API_PROXY_TARGET` or `http://groupscout:8080`.
- Static product assets are generated with `npm run build` from `web/src/renderer/buildStaticApp.js`; the first renderer mapping lives in `web/src/renderer/domRenderer.js`.
- Product dev serving lives in `web/src/server/productDevServer.js`; `compose.dev.yml` runs that server on container port `3000` and host `${GROUPSCOUT_UI_HOST_PORT:-3001}`.
- Backend compatibility smoke classification lives in `web/src/server/backendCompatibilitySmoke.js` and distinguishes proxy failure, backend route drift, auth, schema drift, compatible responses, and backend errors.
- Lead inbox reads use `createApiClient().listLeads(...)` for `GET /api/leads` query serialization, pagination cursors, default priority ordering, and response field adaptation.
- Lead detail reads use `createApiClient().getLead(...)` for `GET /api/leads/{id}` evidence workspace fields, source evidence, AI enrichment, reviewer corrections, and activity rows.
- Lead status writes use `createApiClient().patchLead(...)` for `PATCH /api/leads/{id}` payloads covering status, owner, notes, snooze date, correction reason, and safe field corrections.
- Raw audit reads use `createApiClient().getLeadRawAudit(...)` for the UI-safe `GET /api/leads/{id}/raw` alias.
- Outreach history reads and manual attempt logs use `createApiClient().listLeadOutreach(...)` and `createApiClient().logLeadOutreach(...)` for `GET/POST /api/leads/{id}/outreach`.
- Pipeline history reads and manual run starts use `createApiClient().listPipelineRuns(...)` and `createApiClient().startPipelineRun(...)` for `GET/POST /api/pipeline/runs`.
- Basic analytics reads use `createApiClient().getStats(...)` for `GET /api/stats`.
- Alert reads use `createApiClient().listAlerts(...)` for read-only `GET /api/alerts`.
- System summary reads use `createApiClient().getSystem()` for read-only `GET /api/system`.
- The Today command center lives in `web/src/app/todayCommandCenter.js` and is mounted by `createRouteShell("/")`.
- The Lead Inbox screen model lives in `web/src/app/leadInbox.js` and is mounted by `createRouteShell("/leads")`.
- The Lead Detail Evidence Workspace lives in `web/src/app/leadDetail.js` and is mounted by `createRouteShell("/leads/{id}")`.
- The lead status transition model lives in `web/src/app/leadStatus.js` and keeps valid actions isolated from detail rendering.
- The Verification Queue screen model lives in `web/src/app/verificationQueue.js` and is mounted by `createRouteShell("/verification")`.
- The Outreach Workspace screen model lives in `web/src/app/outreachWorkspace.js` and is mounted by `createRouteShell("/outreach")`.
- The Pipeline Monitor screen model lives in `web/src/app/pipelineMonitor.js` and is mounted by `createRouteShell("/pipeline")`.
- The Analytics screen model lives in `web/src/app/analyticsDashboard.js` and is mounted by `createRouteShell("/analytics")`.
- The Alertd console screen model lives in `web/src/app/alertdConsole.js` and is mounted by `createRouteShell("/alerts")`.
- Phase 2 UI tests cover mocked rows, filters, loading/empty/error states, detail navigation intent, responsive metadata, accessibility metadata, and `DESIGN.md` component-token usage.
- Phase 3 UI tests cover required detail sections, source evidence, raw audit link intent, AI enrichment, reviewer correction distinction, activity timeline entries, loading/not-found/error states, responsive metadata, and `DESIGN.md` component-token usage.
- Phase 4 tests cover the v1 transition table, disallowed action blocking, Lead Detail action visibility, PATCH serialization, required notes/snooze/correction reason validation, and auditable correction payloads.
- Phase 5 tests cover queue trigger classification, verification queue filters/actions, `/verification` route mounting, raw audit API alias use, responsive metadata, token usage, and the blocked redaction-policy TODO.
- Phase 6 tests cover editable outreach drafts, contact validation, manual copied/sent/logged states, outreach API reads/writes, outcome capture, activity timelines, `/outreach` route mounting, responsive metadata, and token usage.
- Phase 7 tests cover pipeline run history, async run creation, collector counts/failures, LLM and delivery health summaries, partial-data states, `/pipeline` route mounting, responsive metadata, and token usage.
- Phase 8 tests cover stats client shape, status/source/score/owner/week summaries, source-yield hit-rate definitions, lead aging, verification quality, demand timing, visible denominator/date-range labels, `/analytics` route mounting, responsive metadata, and token usage.
- Phase 9 tests cover session enforcement for `/api/*`, recursive browser credential exclusion, no automation-token browser headers, `UI_ENABLED`, `UI_BASE_PATH`, deployment readiness, and dev-only CORS configuration.
- Phase 10 tests cover read-only alert state rendering, SPS summaries, evidence, room inventory, action history, disabled mutation actions, `/alerts` route mounting, responsive metadata, token usage, and `GET /api/alerts`.
- Phase 11 tests cover the Today command center, priority lead and aging-work summaries, active alerts, failed jobs, system health, read-only action policy, `/` route mounting, responsive metadata, token usage, and `GET /api/system`.
- Smell Phase H1 split the growing browser API client module while preserving `createApiClient(...)` and the centralized same-origin `/api/*` guard.
- Phase 12 dockerization planning lives in `docs/phase-12-ui-dockerization.md`; the D0-D5 contract lives in `docs/ui-dockerization-contract.md`; the test Docker target runs `npm test`, `compose.dev.yml` adds a development `groupscout-ui` service, the production Docker target serves static assets plus same-origin `/api/*` on container port `3000`, and D5 documents repeatable Docker operations/CI hooks. Phase 13 keeps those boundaries and adds the product renderer/runtime without external dependencies.
- Canonical Phase 0-15 implementation evidence lives in `docs/ui-tdd-phase-0-15-implementation.md`.
- Tests use Node's built-in `node:test` runner so the harness has no package-install requirement yet.

## Test Command

```sh
npm test
```

Static product build:

```sh
npm run build
```

Containerized UI test run:

```sh
docker build --target test -t groupscout-ui-test .
docker run --rm groupscout-ui-test
```

Docker operations:

```sh
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout
curl -i http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/healthz
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml down
```

Backend plus UI Docker smoke:

```sh
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up -d --build groupscout-ui groupscout
curl -i http://localhost:8080/health
curl -i http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/healthz
docker build --target production -t groupscout-ui-production .
docker run --rm -d --name groupscout-ui-production-smoke --network groupscout_groupscout_net -p 3002:3000 -e UI_API_PROXY_TARGET=http://groupscout:8080 groupscout-ui-production
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
docker stop groupscout-ui-production-smoke
```

The merged Compose service on port `3001` now runs the Phase 13 product dev server. It proves backend-network wiring, serves the static product assets, and preserves server-side backend discovery. The production container on port `3002` remains the same-origin static/proxy runtime. Current backend docs and smoke scripts expect `/api/leads?limit=1` to reach the backend through the UI production proxy when `UI_SESSION_SECRET` is not configured. When `UI_SESSION_SECRET` is configured for a real operator deployment, unauthenticated `/api/*` requests return `401` before proxying.

Production UI runtime:

```sh
npm run start:ui
docker build --target production -t groupscout-ui-production .
docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
```

The default development UI host port is `3001` because the backend stack publishes Grafana on `3000`; set `GROUPSCOUT_UI_HOST_PORT` to use another host port. The D3 Compose smoke path starts `groupscout-ui` with backend service `groupscout`; the backend dependency chain also starts `postgres`, `ollama`, and `ollama-init`. Do not run UI Docker containers with backend `.env` or `--env-file`, and do not pass `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` into browser-visible config, static assets, Compose output, or CI artifacts.

## Developer Docs

- [Developer Guide](./docs/developer-guide.md)
- [How To Run The Backend](./docs/how-to-run-backend.md)
- [Testing](./docs/testing.md)
- [Troubleshooting](./docs/troubleshooting.md)
- [Nice To Knows](./docs/nice-to-knows.md)
- [Code Smells And Housekeeping Notes](./docs/code-smells.md)
- [Code Smell Transformation TDD Prompts](./docs/code-smell-transformation-prompts.md)
- [Smell H0 API Client Characterization](./docs/smell-h0-api-client-characterization.md)
- [Smell H1 API Client Split](./docs/smell-h1-api-client-split.md)
- [Phase 0 Product Contract](./docs/phase-0-product-contract.md)
- [Phase 1 Lead Inbox Contract](./docs/phase-1-lead-inbox-contract.md)
- [Phase 2 Lead Inbox UI](./docs/phase-2-lead-inbox-ui.md)
- [Phase 3 Lead Detail Evidence Workspace](./docs/phase-3-lead-detail-evidence-workspace.md)
- [Phase 4 Lead Status Actions And State Model](./docs/phase-4-lead-status-actions-state-model.md)
- [Phase 5 Verification Queue And Raw Audit Review](./docs/phase-5-verification-queue-raw-audit-review.md)
- [Phase 6 Outreach Workspace And Activity Log](./docs/phase-6-outreach-workspace-activity-log.md)
- [Phase 7 Pipeline Monitor And Run Controls](./docs/phase-7-pipeline-monitor-run-controls.md)
- [Phase 8 Basic Analytics And Demand Signals](./docs/phase-8-basic-analytics-demand-signals.md)
- [Phase 9 Session/Auth Wrapper And Same-Origin Deployment](./docs/phase-9-session-auth-wrapper-same-origin-deployment.md)
- [Phase 10 Later Alertd Read-Only Console](./docs/phase-10-later-alertd-read-only-console.md)
- [Phase 11 Today Command Center And System Health Summary](./docs/phase-11-today-command-center-system-health.md)
- [Phase 12 UI Dockerization Prompt Pack](./docs/phase-12-ui-dockerization.md)
- [UI Dockerization Contract](./docs/ui-dockerization-contract.md)
- [Docker Runtime Matrix](./docs/docker-runtime-matrix.md)
- [Phase 13 Product Renderer Runtime](./docs/phase-13-product-renderer-runtime.md)
- [Phase 13 Product Renderer Runtime Prompt Pack](./docs/phase-13-product-renderer-runtime-prompts.md)
- [UI TDD Phases 0-15 Implementation Status](./docs/ui-tdd-phase-0-15-implementation.md)
- [Phase 15 Browser UX Hardening](./docs/phase-15-browser-ux-hardening.md)

## Phase 0 Guardrails

- Browser-facing source must not reference automation credentials.
- Browser requests must stay behind explicit `/api/*` contracts.
- Production `/api/*` proxy requests require a valid `groupscout_session` before the backend is contacted when `UI_SESSION_SECRET` is configured; the backend Docker smoke path intentionally leaves the session secret unset so proxy reachability can be checked without browser credentials.
- The shell reserves navigation for Today, Leads, Verification, Outreach, Pipeline, Analytics, Alerts, and Settings.
- Historically, Phase 0 kept feature screens, real API calls, auth, analytics, and deployment behavior out of scope. Later phases now add model-level screens, browser API clients, analytics, session/deployment metadata, and read-only system surfaces.

## Phase 1 Lead Inbox Contract

- Endpoint: `GET /api/leads`.
- Filters: `q`, `status`, `source`, `min_score`, `created_from`, `created_to`, `property`, `owner`, `verification_state`, `limit`, and `cursor`.
- Default sort: `priority`, representing urgent, unowned, high-score leads first with newest leads as the final tiebreaker.
- Response adapter fields: score, title, segment/project type, location/property fit, source, crew/duration estimate, outreach timing, status, owner, created date, evidence state, and verification state.

## Phase 2 Lead Inbox UI

- Screen model: `createLeadInboxScreen(...)`.
- Controls: search, status, source, minimum score, created date range, property, owner, verification, and clear filters.
- Table columns: score, title, segment/project, location/property, source, crew/duration, outreach timing, status, owner, created, and evidence/verification.
- Responsive behavior: desktop table, tablet priority table, and mobile lead cards.
- Out of scope: status mutations, outreach logging, raw audit viewing, lead detail evidence, and analytics.

## Phase 3 Lead Detail Evidence Workspace

- Screen model: `createLeadDetailScreen(...)`.
- Required sections: Summary, Source Evidence, AI Enrichment, Actions, Outreach, and Activity.
- Summary fields: title, score, timing, room-night signal, and property fit.
- Source Evidence fields: source name, source URL, raw audit link intent, and collected timestamp.
- AI Enrichment fields: contractor/applicant, project type, crew size, duration, rationale, uncertainty, and per-claim source evidence.
- Reviewer corrections are shown alongside original AI extraction values and never silently replace source-backed values.
- Responsive behavior: desktop evidence workspace, tablet evidence stack, and mobile detail stack with actions reachable.
- Raw audit links use the UI-safe `/api/leads/{id}/raw` alias.
- Out of scope: outreach logging and inline raw audit payload rendering.

## Phase 4 Lead Status Actions

- Statuses: `new`, `notified`, `claimed`, `contacted`, `snoozed`, `flagged`, `verified`, `dismissed`, `won`, `lost`, and `no_response`.
- Actions are generated from the transition table so operators only see valid actions for the current status.
- Invalid actions are rejected before `patchLead(...)` is called.
- `follow_up` and `corrected` are modeled as actions, not statuses, and preserve the current status.
- Corrections preserve original AI/source values and include actor plus reason metadata before API serialization.

## Phase 5 Verification Queue And Raw Audit Review

- Screen model: `createVerificationQueueScreen(...)`.
- Queue triggers: missing source/raw audit, high score with weak rationale, raw/enriched contradiction, low confidence collector parse, and manual operator flag.
- Controls: trigger, source, owner, minimum score, and clear filters.
- Row actions: verify, correct, dismiss, and return to lead.
- Raw audit review links and client reads use `GET /api/leads/{id}/raw`.
- Redaction policy remains explicitly blocked until raw payload redaction rules are defined.

## Phase 6 Outreach Workspace And Activity Log

- Screen model: `createOutreachWorkspaceScreen(...)`.
- Lead Detail embeds a compact outreach workspace for the selected lead.
- Editable fields: channel, contact details, draft/message, notes, and outcome.
- Manual display states: draft ready, copied, marked sent, and logged.
- Supported outcomes: `contacted`, `won`, `lost`, and `no_response`.
- API boundary: `GET /api/leads/{id}/outreach` for history and `POST /api/leads/{id}/outreach` for manual attempt logs.
- Activity rows distinguish manual outreach attempts and outcomes from source evidence, AI enrichment, and reviewer corrections.
- Out of scope: automated sending, CRM sync, bulk outreach, and analytics.

## Phase 7 Pipeline Monitor And Run Controls

- Screen model: `createPipelineMonitorScreen(...)`.
- Health fields: last run/result, collector collected/skipped/enriched counts, collector failures, LLM provider/latency/errors, and Slack/email/webhook delivery failures.
- API boundary: `GET /api/pipeline/runs` for recent history and `POST /api/pipeline/runs` for manual run creation.
- Manual run creation is modeled as accepted/queued browser-async work; the UI does not wait for the whole pipeline to finish.
- Logs and Grafana are optional display links when the backend supplies them.
- Out of scope: replacing Grafana, polling worker internals directly, exposing automation-only endpoints, and building a full observability dashboard.

## Phase 8 Basic Analytics And Demand Signals

- Screen model: `createAnalyticsDashboardScreen(...)`.
- API boundary: `GET /api/stats` for date-range, segment, and property filtered stats.
- Summary groups: status, source, score band, owner, and week.
- Source yield defines hit rate as `won leads / total source leads`, with denominator and date range visible.
- Operational views: lead aging, verification quality, and upcoming demand by week, segment, and property.
- Out of scope: custom dashboard builder, direct database access, CRM replacement reporting, and unexplained metrics.

## Phase 9 Session/Auth Wrapper And Same-Origin Deployment

- Deployment helper: `createUiDeploymentConfig(...)`, `assertUiDeploymentReady(...)`, `resolveUiMount(...)`, and `authorizeUiApiRequest(...)`.
- Session cookie: `groupscout_session` is required for browser `/api/*` access when the UI is enabled and `UI_SESSION_SECRET` is configured.
- Settings: `UI_ENABLED`, `UI_BASE_PATH`, `UI_SESSION_SECRET`, and development-only `CORS_ALLOWED_ORIGINS`.
- Mounted shell: `createMountedRouteShell(...)` maps URLs under `UI_BASE_PATH` to internal routes and emits base-path-aware hrefs.
- Out of scope: role matrices, identity-provider UI, production proxy config, and repurposing `API_TOKEN` for browser sessions.

## Phase 10 Later Alertd Read-Only Console

- Screen model: `createAlertdConsoleScreen(...)`.
- API boundary: read-only `GET /api/alerts` for state, property, limit, and cursor filtered alert data.
- Summary fields: active alert count, highest SPS, critical unavailable-room impact, and last update.
- Detail fields: evidence rows, room inventory, and action history.
- Slack remains the interrupt channel; acknowledge, resolve, and suppress actions stay disabled and out of scope.

## Phase 11 Today Command Center And System Health Summary

- Screen model: `createTodayCommandCenterScreen(...)`.
- API boundary: read-only `GET /api/system` for UI-friendly system health, pipeline freshness, and operational counts.
- Summary fields: high-score new leads, aging claimed leads, active alerts, failed jobs, and overall system health.
- Work sections link to the owning screens: Leads, Verification, Outreach, Pipeline, and Alerts.
- Today is read-only; status, outreach, alert, pipeline, and settings mutations stay out of scope.
