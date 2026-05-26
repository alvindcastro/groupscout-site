## 2026-05-25 - Missed task markdown reconciliation

### groupscout-site-1t9 - Enable Sunday Wednesday n8n cadence

- **What:** Wired the Compose n8n service to the GroupScout API and Slack environment variables, re-imported the corrected cadence workflow body, then activated the Sunday/Wednesday cadence workflow in the live n8n container.
- **Where:** Backend `docker-compose.yml`, live `groupscout_n8n`, and centralized n8n setup/troubleshooting docs.
- **When:** 2026-05-25.
- **Why:** The workflow schedule was correct, but it could not reliably send while inactive, missing the env values used by `$env.*` expressions, and running a stale plain `/run` body.
- **How:** Recreated n8n with `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, `GROUPSCOUT_API_BASE_URL`, `GROUPSCOUT_API_TOKEN`, and `GROUPSCOUT_OPS_SLACK_WEBHOOK_URL`; re-imported the tracked workflow; verified backend health, secret-safe env presence, live `active:true`, Sunday/Wednesday 09:00 `America/Vancouver` schedule fields, and `guarantee_one_lead`/`delivery_mode`/`idempotency_key` body fields.

---

### groupscout-site-9fy - Audit backend markdown for missed task drift

- **What:** Reconciled backend markdown that still presented planned contact enrichment, health/OpenAPI follow-ups, and older strategic checklists as live or unowned work.
- **Where:** Backend architecture/data-flow docs, roadmap and phase planning docs, UI API endpoint notes, AI/Ollama planning docs, and this changelog.
- **When:** 2026-05-25.
- **Why:** The checked backend source and centralized docs contradicted each other around Hunter.io contact enrichment, smoke-artifact branch history, health OpenAPI drift, and unchecked follow-up ownership.
- **How:** Cross-checked existing open Beads, current backend source filenames, and backend markdown references; added factual source-snapshot caveats and existing bead IDs without changing source code or closing work.

---

### groupscout-site-cw2 - Review MD files for missed task drift

- **What:** Updated markdown task state around the completed `/ingest` work, live missed-task follow-ups, and UI-planning endpoint inventories.
- **Where:** Centralized backend README, roadmap/phase/future-integration docs, UI API planning docs, and the legacy Phase 25 prompt archive.
- **When:** 2026-05-25.
- **Why:** The docs still had stale planned-language for raw single-item enrichment and omitted the remaining Postgres FK follow-up from the live missed-task list.
- **How:** Cross-checked Beads and parallel agent findings, then aligned endpoint tables, Phase 26 labels, and handoff notes with `groupscout-site-b25` complete and `groupscout-site-wda` open.

---

## 2026-05-25 - Guaranteed lead cadence

### groupscout-site-fuc - Guaranteed Sunday/Wednesday one-lead delivery

- **What:** Added backend delivery semantics for exactly one eligible cadence lead.
- **Where:** Backend delivery log, run lock, fallback selector, and `/run` JSON result behavior; centralized setup, n8n, testing, troubleshooting, and API docs.
- **When:** 2026-05-25.
- **Why:** The Sunday/Wednesday operator prompt needs a dependable single lead without duplicate Slack sends on retries.
- **How:** Documented `lead_deliveries`, `delivery_locks`, cadence/idempotency keys, `delivery_status` outcomes, and backlog fallback behavior.

### groupscout-site-yfl - Importable Sunday/Wednesday n8n workflow

- **What:** Added the importable n8n workflow asset and Docker import smoke notes for the guaranteed cadence.
- **Where:** `backend/docs/workflows/n8n/sunday-wednesday-lead-cadence.json`, workflow README, and setup/runbook docs.
- **When:** 2026-05-25.
- **Why:** Operators need the Sunday/Wednesday workflow to be reproducible rather than recreated by hand.
- **How:** Captured the schedule, health preflight, guaranteed `/run` request, cadence idempotency key, retry behavior, and ops Slack no-leads/failure branch.

---

## 2026-05-25 - Event-driven ingest endpoint

### groupscout-site-b25 - Implement event-driven ingest endpoint

- **What:** Added a backend `POST /ingest` path for one raw project/event payload plus `EnrichOne()` on the enrichment pipeline.
- **Where:** Backend source branch `task/event-driven-ingest`; centralized backend API, n8n, roadmap, and setup docs.
- **When:** 2026-05-25.
- **Why:** n8n and other automation sources need to push one raw item through GroupScout enrichment without triggering the full collector pipeline or direct-inserting a pre-enriched lead.
- **How:** Added failing single-item enrichment and HTTP handler tests first, normalized inbound raw project payloads, reused the existing dedup/audit/score/enrich/store path, protected `/ingest` with the existing bearer-token rule, updated OpenAPI, and documented when to use `/ingest` versus `/n8n/webhook`.

---

## 2026-05-25 - Source snapshot reconciliation

### groupscout-site-6s9 - Review missed task documentation drift

- **What:** Marked several backend/UI API, admin auth, smoke, EvalOps, and frontend hardening entries as planned or branch-history where the inspected source checkouts do not currently contain the referenced implementation files.
- **Where:** Centralized backend and frontend Markdown docs in `backend/` and `frontend/`.
- **When:** 2026-05-25.
- **Why:** The current backend source snapshot exposes only the automation/server routes plus the legacy raw-audit route, and the current UI checkout lacks several admin/session, detail-client, and Phase 15 hardening artifacts described by the docs.
- **How:** Cross-checked Beads, backend source, frontend source, and docs with parallel agents; kept historical changelog entries intact but added current-source caveats and follow-up links.

Historical 2026-05-09 entries for admin auth, `/api/*` operator routes, EvalOps, and UI Docker smoke describe branch-history work. They are not current-source proof until `groupscout-site-eqm`, `groupscout-site-crz`, and related reconciliation tasks restore or merge those files into the inspected backend checkout.

---

## 2026-05-25 - Raw audit retention worker

### groupscout-site-j73 - Implement raw audit retention purge worker

- **What:** Added the opt-in backend raw audit retention worker, manual purge command, Docker environment controls, and Postgres purge-safety coverage.
- **Where:** Backend source `internal/auditretention`, `cmd/server`, `config`, Docker env, and centralized backend setup/testing/Postgres/audit roadmap docs.
- **When:** 2026-05-25.
- **Why:** Raw audit payloads need bounded storage growth without deleting evidence still referenced by leads.
- **How:** Reused `AuditStore.PurgeOlderThan`, preserved every `leads.raw_input_id` reference, exposed `AUDIT_RETENTION_*` configuration, and documented the Docker/operator flow.

---

## 2026-05-25 - Documentation consolidation

### groupscout-site-fjy - Consolidate backend UI-testing runbook routing

- **What:** Trimmed duplicate backend-plus-UI Docker smoke commands from the local backend-for-UI testing runbook.
- **Where:** `docs/planning/ui/BACKEND_FOR_UI_TESTING.md`.
- **When:** 2026-05-25.
- **Why:** The same Compose override, UI dev-server, D4 production proxy, health curl, and troubleshooting expectations were duplicated between the local backend startup guide and the full backend-plus-frontend Docker E2E runbook.
- **How:** Kept `BACKEND_FOR_UI_TESTING.md` focused on starting a backend for UI work and linked the Docker smoke section to `BACKEND_FRONTEND_DOCKER_E2E.md` as the canonical runbook.

---

## 2026-05-09 - Admin token flow documentation

### groupscout-site-0o7 - Document admin token login flow

- **What:** Added a dedicated backend admin auth guide and expanded centralized backend/frontend docs for setup-token login, cookie sessions, logout, token rotation, Docker smoke, and stale-asset recovery.
- **Where:** `docs/guides/ADMIN_AUTH.md`, `docs/API_CONFIG.md`, `docs/guides/DOCKER.md`, `docs/guides/TESTING.md`, `docs/guides/TROUBLESHOOTING.md`, `docs/DEVELOPER.md`, `docs/planning/ui/BACKEND_FOR_UI_TESTING.md`, `docs/planning/ui/UI_API_ENDPOINTS.md`, and frontend centralized docs.
- **When:** 2026-05-09.
- **Why:** Operators need a single clear flow for finding the setup token, logging in, understanding token rotation, logging out, and diagnosing stale Docker/browser state.
- **How:** Documented `ADMIN_AUTH_ENABLED`, `ADMIN_SETUP_TOKEN`, `ADMIN_SETUP_TOKEN_FILE`, `ADMIN_SESSION_TTL_HOURS`, `groupscout_session`, bearer session-token smoke usage, and the `admin-login-1` static asset cache key.

---

## 2026-05-09 - Docker testing prep documentation

### groupscout-site-1fr - Refresh testing Docker documentation

- **What:** Documented the current Docker testing-ready state, including the spun-up Postgres, backend API, UI product dev server, n8n, and Ollama caveat.
- **Where:** `docs/guides/TESTING.md`, `docs/guides/DOCKER.md`, `docs/guides/TROUBLESHOOTING.md`, `docs/DEVELOPER.md`, and related frontend centralized docs.
- **When:** 2026-05-09.
- **Why:** Manual testing needs a clear split between lightweight UI/API smoke, Postgres-backed integration tests, full Docker E2E, and LLM/enrichment checks.
- **How:** Confirmed the running Docker services, verified backend and UI smoke endpoints, refreshed the testing/how-to/developer/troubleshooting/nice-to-know docs, and recorded that backend `/health` can be `200` with database OK while Ollama remains unavailable.

---

## 2026-05-09 - Admin login session completion

### groupscout-site-9ge - Complete backend admin login flow

- **What:** Hardened the setup-token admin login flow with logout/revoke support, file-backed setup-token rotation after successful login, session expiry tests, bearer session-token validation, OpenAPI/Bruno auth coverage, and authentication for the legacy `/leads/{id}/raw` route.
- **Where:** `cmd/server/admin_auth.go`, `cmd/server/main.go`, `cmd/server/ui_api.go`, `cmd/server/ui_api_test.go`, `api/swagger.yaml`, `api/bruno/AdminAuthStatus.bru`, `api/bruno/AdminLogin.bru`, `api/bruno/AdminLogout.bru`, `api/bruno/environments/Local.bru`, and centralized docs.
- **When:** 2026-05-09.
- **Why:** The first admin flow could issue a session, but it had no logout path, reusable file-backed setup tokens, weak test coverage for session lifecycle behavior, missing API docs, and a legacy raw-audit route that could bypass the UI admin session boundary.
- **How:** Added `POST /api/auth/logout`, session revocation, expired cookie clearing, file-backed setup-token rotation, legacy raw route auth checks, tests for invalid login, `/api/auth/me`, bearer session auth, expiry, logout, token reuse rejection, and raw-route auth, then documented the auth endpoints and Bruno requests.

---

## 2026-05-09 - Frontend Docker contract compatibility

### groupscout-ehq - Review frontend UI docs and Docker contracts

- **What:** Added backend compatibility for the frontend-documented read-only `GET /api/alerts` route and tightened the backend-owned UI Docker smoke expectations now that `/api/system` is implemented.
- **Where:** `cmd/server/ui_api.go`, `cmd/server/ui_api_phase37_test.go`, `scripts/smoke-ui-docker-e2e.sh`, `internal/smoke/ui_docker_script_test.go`, `api/swagger.yaml`, `docs/API_CONFIG.md`, `docs/planning/ui/UI_API_ENDPOINTS.md`, `docs/planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md`, `docs/planning/ui/BACKEND_FOR_UI_TESTING.md`, `docs/planning/ui/UI_PHASE38_DOCKER_SMOKE_CONTRACT.md`, `docs/guides/TESTING.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** The external `groupscout-ui` markdown and Docker files model `/api/alerts` as a same-origin read-only route, while the backend only exposed leads, outreach, pipeline runs, stats, and system routes. The Docker smoke also still allowed old `/api/system` route drift.
- **How:** Inspected the UI repo Dockerfile, Compose override, API client docs, and alert client tests; wrote a failing backend HTTP test for `/api/alerts`; implemented an empty read-only compatibility response with `alerts` plus `items` arrays; added `/api/alerts` to OpenAPI/docs; and updated the Docker smoke script to require `/api/system` and `/api/alerts` through the UI production proxy.

---

## 2026-05-09 - Phases 36-38 UI API and Docker smoke contracts

### groupscout-al3 - Implement Phase 36 outreach and lead state API contracts

- **What:** Added outreach log storage, `/api/leads/{id}/outreach` GET/POST, schema-backed lead workflow fields, and validated lead actions for claim, dismiss, snooze, flag, contacted, won, lost, no-response, and reopen.
- **Where:** `internal/storage/outreach.go`, `internal/storage/outreach_test.go`, `internal/storage/leads.go`, `internal/storage/db.go`, `cmd/server/ui_api.go`, `cmd/server/ui_api_phase36_test.go`, `migrations/008_ui_workflow_fields.up.sql`, `migrations/008_ui_workflow_fields.down.sql`, `api/swagger.yaml`, `docs/planning/ui/UI_PHASE36_OUTREACH_STATE_CONTRACT.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/planning/ui/UI_API_ENDPOINTS.md`, `docs/API_CONFIG.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** Operators need durable outreach history and constrained workflow actions instead of free-form status edits before the UI can safely support lead triage and follow-up.
- **How:** Wrote failing storage and HTTP tests first, added `OutreachStore`, added workflow fields for owner, snooze, flag, and verification separation, implemented action validation in storage, routed outreach GET/POST, returned `409` for invalid transitions, and documented the contract.

### groupscout-0pm - Implement Phase 37 pipeline runs stats and system API contracts

- **What:** Added persisted pipeline run tracking, nonblocking `POST /api/pipeline/runs`, `GET /api/pipeline/runs`, `GET /api/stats`, and `GET /api/system`.
- **Where:** `internal/storage/pipeline_runs.go`, `internal/storage/pipeline_runs_test.go`, `internal/storage/stats.go`, `internal/storage/db.go`, `cmd/server/ui_api.go`, `cmd/server/ui_api_phase37_test.go`, `cmd/server/main.go`, `migrations/009_pipeline_runs.up.sql`, `migrations/009_pipeline_runs.down.sql`, `api/swagger.yaml`, `docs/planning/ui/UI_PHASE37_PIPELINE_STATS_SYSTEM_CONTRACT.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/planning/ui/UI_API_ENDPOINTS.md`, `docs/API_CONFIG.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** The operator UI needs nonblocking pipeline execution, run history, dashboard stats, and health state without scraping Prometheus metrics in browser code.
- **How:** Wrote failing storage and HTTP tests first, added a `pipeline_runs` table and store, wired production `/api/pipeline/runs` to run the real pipeline asynchronously, added stats queries over supported schema fields, exposed `/api/system`, and documented browser-safe health semantics.

### groupscout-kh1 - Implement Phase 38 Docker UI smoke checks

- **What:** Added a backend-owned Docker smoke script for the external production UI container and a Go test that locks the smoke script contract.
- **Where:** `scripts/smoke-ui-docker-e2e.sh`, `internal/smoke/ui_docker_script_test.go`, `docs/planning/ui/UI_PHASE38_DOCKER_SMOKE_CONTRACT.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/guides/TESTING.md`, `README.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** Phase 38 needs a repeatable way to verify backend Compose plus the separate UI production static/proxy runtime, same-origin API behavior, backend 404 versus proxy 502 semantics, route smoke, and browser-visible secret safety.
- **How:** Wrote a failing script-contract test first, added `scripts/smoke-ui-docker-e2e.sh` with Compose startup, production UI container checks for `/`, `/healthz`, and `/assets/app.js`, proxy checks for `/api/leads` and bad-target `502`, UI repo Node route smoke, static secret scans, and cleanup traps.

---

## 2026-05-09 - Phase 35 UI API contracts

### groupscout-pvx - Implement UI lead API contracts with strict TDD

- **What:** Added the first implemented same-origin UI lead API contracts: filtered lead listing, lead detail with audit metadata, safe lead patching, authenticated raw audit access, OpenAPI updates, and temporary frontend type documentation.
- **Where:** `internal/storage/leads.go`, `internal/storage/leads_test.go`, `cmd/server/ui_api.go`, `cmd/server/ui_api_test.go`, `cmd/server/main.go`, `api/swagger.yaml`, `docs/planning/ui/UI_PHASE35_API_CONTRACT.md`, `docs/planning/ui/UI_PHASE35_FRONTEND_TYPES.md`, `docs/prompts/PROMPTS_PHASE35_UI.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/planning/ui/UI_API_ENDPOINTS.md`, `docs/planning/ui/README.md`, `docs/API_CONFIG.md`, `docs/guides/TESTING.md`, `README.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** Phase 35 needed live backend contracts for the planned operator UI so the Phase 33 inbox and Phase 34 detail boundaries can move from fixtures to typed `/api/*` data without exposing raw payload bodies or unsafe source-backed edits.
- **How:** Wrote failing storage and HTTP tests first, implemented `LeadListFilter`/`ListFiltered`, added a testable `/api/leads` handler, mapped storage leads to snake_case safe response DTOs, allowed only `status` and `notes` in `PATCH`, required bearer auth for raw evidence when `API_TOKEN` is configured, updated OpenAPI after tests defined behavior, documented schema-backed limits for owner, snooze, verification, and correction fields, and filed `groupscout-al3` for Phase 36 outreach/state endpoints.

---

### groupscout-44x - Fix OpenAPI health response contract drift

- **What:** Corrected the OpenAPI `/health` response contract from plain text to JSON.
- **Where:** `api/swagger.yaml` and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** The implemented health handler returns JSON with `status`, `database`, and `ollama`, so generated clients should not expect a text/plain `OK` body.
- **How:** Added the shared `HealthStatus` schema and documented both `200` and `503` health responses as `application/json`.

---

## 2026-05-09 - Phase 34 lead detail and evidence review contract

### groupscout-ftx - Implement Phase 34 lead detail evidence review contract

- **What:** Added the Phase 34 lead detail and evidence review implementation contract plus a dedicated Phase 34 prompt pack for strict TDD detail fixture, summary, source evidence, AI rationale, raw audit safety, outreach/activity, status action, correction-control, responsive, build, and static-asset verification work.
- **Where:** `docs/planning/ui/UI_PHASE34_LEAD_DETAIL_EVIDENCE_CONTRACT.md`, `docs/prompts/PROMPTS_PHASE34_UI.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/planning/ui/README.md`, `docs/planning/ui/UI_DESIGN_SYSTEM.md`, `docs/planning/ui/UI_STRATEGY.md`, `docs/planning/ui/UI_API_ENDPOINTS.md`, `docs/prompts/PROMPTS_PHASE31_UI.md`, `docs/planning/ROADMAP.md`, `docs/ARCHITECTURE.md`, `README.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** Phase 34 needed enforceable lead detail and evidence-review acceptance criteria before a frontend package or live `GET /api/leads/{id}` route exists, so future UI work starts from failing tests and deterministic fixtures instead of untested detail screens or accidental raw-payload loading.
- **How:** Documented the fixture-only detail boundary, required detail and audit metadata fields, source-evidence adjacency rules, missing-evidence/weak-confidence/contradiction states, disabled or mocked correction controls, fixture-only outreach/activity and status actions, raw audit link behavior without default payload bodies, accessibility/responsive expectations, replaceable `/api/leads/{id}` client boundary, and static secret-scan expectations, then cross-linked the new contract from the UI docs, prompt docs, architecture, roadmap, README, and TDD phase plan.

---

## 2026-05-09 - Phase 33 mocked lead inbox contract

### groupscout-aaf - Implement Phase 33 mocked lead inbox contract

- **What:** Added the Phase 33 mocked lead inbox implementation contract plus a dedicated Phase 33 prompt pack for strict TDD fixture, table, search, filter, sorting, keyboard, responsive, build, and static-asset verification work.
- **Where:** `docs/planning/ui/UI_PHASE33_MOCKED_LEAD_INBOX_CONTRACT.md`, `docs/prompts/PROMPTS_PHASE33_UI.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/planning/ui/README.md`, `docs/planning/ui/UI_STRATEGY.md`, `docs/planning/ui/UI_DESIGN_SYSTEM.md`, `docs/prompts/PROMPTS_PHASE31_UI.md`, `docs/planning/ROADMAP.md`, `docs/ARCHITECTURE.md`, `README.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** Phase 33 needed enforceable lead inbox acceptance criteria before a frontend package or live `GET /api/leads` route exists, so future UI work starts from failing tests and deterministic fixtures instead of untested browser code or accidental automation endpoint coupling.
- **How:** Documented the fixture-only data boundary, required lead summary fields, table columns, loading/empty/error/populated states, search and filter behavior, default sorting, keyboard navigation, accessible controls, replaceable `/api/leads` client boundary, raw-audit exclusion, and static secret-scan expectations, then cross-linked the new contract from the UI docs, prompt docs, architecture, roadmap, README, and TDD phase plan.

---

## 2026-05-09 — Phase 32 UI app shell and routing contract

### groupscout-swx — Implement Phase 32 app shell routing contract

- **What:** Added the Phase 32 app shell and routing implementation contract plus a dedicated Phase 32 prompt pack for strict TDD route, layout, responsive, build, component, and static-asset verification work.
- **Where:** `docs/planning/ui/UI_PHASE32_APP_SHELL_ROUTING_CONTRACT.md`, `docs/prompts/PROMPTS_PHASE32_UI.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/planning/ui/README.md`, `docs/planning/ui/UI_STRATEGY.md`, `docs/planning/ui/UI_DESIGN_SYSTEM.md`, `docs/prompts/PROMPTS_PHASE31_UI.md`, `docs/planning/ROADMAP.md`, `docs/ARCHITECTURE.md`, `README.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** Phase 32 needed enforceable app-shell and routing acceptance criteria before a frontend package exists, so future UI code starts from failing tests for the operator workspace instead of untested shell scaffolding.
- **How:** Documented the no-frontend current scope, route inventory for `/`, `/today`, `/leads`, `/leads/:leadId`, `/verification`, `/pipeline`, `/settings`, shell landmarks, right-rail capacity, responsive collapse rules, no-hero guardrails, build and static secret-scan expectations, then cross-linked the contract from the UI docs, roadmap, prompts, architecture, and README.

---

## 2026-05-09 — Phase 31 UI design system contract

### groupscout-h80 — Implement Phase 31 UI design system adaptation docs

- **What:** Added the Phase 31 design-system implementation contract for GroupScout operator UI tokens, primitives, component tests, accessibility checks, static asset secret safety, and no-marketing-hero guardrails.
- **Where:** `docs/planning/ui/UI_PHASE31_DESIGN_SYSTEM_CONTRACT.md`, `docs/planning/ui/UI_DESIGN_SYSTEM.md`, `docs/planning/ui/UI_TDD_PHASE_PLAN.md`, `docs/planning/ui/README.md`, `docs/planning/ui/UI_STRATEGY.md`, `docs/prompts/PROMPTS_PHASE31_UI.md`, `docs/ARCHITECTURE.md`, `README.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-09.
- **Why:** Phase 31 needed to translate the external `groupscout-ui/DESIGN.md` reference into enforceable GroupScout operator UI rules before a frontend package exists in this repository.
- **How:** Read the external design reference, kept the dense documentation/product-surface language, excluded marketing hero patterns, documented token and primitive acceptance, added required future failing test coverage, and specified a browser-visible secret scan for future static assets.

---

## 2026-05-09 — Backend and frontend Docker E2E docs

### groupscout-hi7 — Document backend plus UI Docker smoke path

- **What:** Added a docs-only runbook for running the backend Compose stack with the separate UI Docker image, including the D3 health harness and D4 production static/proxy smoke path.
- **Where:** `docs/planning/ui/BACKEND_FRONTEND_DOCKER_E2E.md`, `docs/planning/ui/BACKEND_FOR_UI_TESTING.md`, `docs/planning/ui/UI_STRATEGY.md`, `docs/planning/ui/README.md`, `docs/guides/DOCKER.md`, `docs/DEVELOPER.md`, `docs/guides/TESTING.md`, and `docs/guides/TROUBLESHOOTING.md`.
- **When:** 2026-05-09.
- **Why:** Developers need a clear way to run backend and frontend together in Docker without confusing the UI D3 health harness with the D4 static/proxy runtime.
- **How:** Documented stable `-p groupscout` Compose commands, ports `8080`/`3001`/`3002`, `groupscout_groupscout_net`, `UI_API_PROXY_TARGET=http://groupscout:8080`, expected `404` for not-yet-implemented UI `/api/*` routes, and `502` as the proxy/network failure signal.

---

## 2026-05-08 - Beads prompt workflow guide

### groupscout-3jw - Document Beads prompt workflow

- **What:** Added a guide for converting Markdown prompt habits into Beads-backed task tracking.
- **Where:** `docs/guides/BEADS_WORKFLOW.md` and `docs/CHANGELOG.md`.
- **When:** 2026-05-08.
- **Why:** Prompt-heavy workflows need a clear split between reusable Markdown context and live Beads task state.
- **How:** Documented command flows, prompt-to-bead mapping, parallel agent guidance, changelog expectations, and commit-message practices.

---

## 2026-05-08 — Backend runbook for UI testing

### UI-T02 — Document backend startup paths for UI work

- **What:** Added a UI-focused backend runbook with local SQLite, local Postgres, and full Docker startup paths, plus health checks, useful API calls, frontend proxy notes, current endpoint status, and troubleshooting.
- **Where:** `docs/planning/ui/BACKEND_FOR_UI_TESTING.md`, `docs/planning/ui/README.md`, and `README.md`.
- **When:** 2026-05-08.
- **Why:** Frontend work needs a clear way to spin up the backend without assuming the planned `/api/*` UI endpoints already exist.
- **How:** Documented the lightweight `go run` path for UI-only testing, the Postgres-backed path for persistence behavior, and the heavier Docker path for n8n, observability, and Ollama-backed testing.

---

## 2026-05-08 — Operator UI strategy

### UI-T01 — Define operator UI strategy and integration plan

- **What:** Added the planning source of truth for the future GroupScout operator UI, including user jobs, workflow, information architecture, lead state model, frontend architecture, UI API contracts, MVP scope, and open decisions.
- **Where:** `docs/planning/ui/UI_STRATEGY.md`, `docs/planning/ui/README.md`, `README.md`, `docs/ARCHITECTURE.md`, `docs/DATA_FLOW.md`, `docs/API_CONFIG.md`, `docs/planning/ROADMAP.md`, and `docs/planning/PHASES.md`.
- **When:** 2026-05-08.
- **Why:** Phase 10 needed concrete UI direction before implementation so Slack, the future web UI, raw audit review, outreach logging, and analytics have clear boundaries.
- **How:** Synthesized parallel UI strategy, architecture/integration, and operator-flow reviews into a single UI planning document and concise cross-links in the existing architecture, data-flow, API, roadmap, and phase-tracker docs.

---

## 2026-05-07 — GQ5 Feedback loop

### GQ5-T01 — Define production-to-case workflow

- **What:** Documented the path from redacted production review sample to reviewed golden JSONL case.
- **Where:** `README.md`, `docs/planning/AI_QUALITY_EVALOPS.md`, `docs/evals/groupscout-case-schema.md`, `data/evals/groupscout/README.md`, `docs/guides/VERIFICATION.md`, and `docs/guides/TESTING.md`.
- **When:** 2026-05-07.
- **Why:** Production failures need a repeatable owner-reviewed route into regression coverage without leaking source data or bypassing human expected-outcome review.
- **How:** Added capture, triage, owner-note, draft generation, TODO replacement, promotion, fixture-count, and release-gate steps.

### GQ5-T02 — Code: sample-to-case helper

- **What:** Added a helper that converts redacted `ReviewSample` entries into review-only draft eval cases.
- **Where:** `internal/evalops/draft_case.go` and `internal/evalops/draft_case_test.go`.
- **When:** 2026-05-07.
- **Why:** GQ4 review samples must be reusable as future eval cases, but generated cases must not become release-blocking or golden without human review.
- **How:** Implemented `DraftCasesFromReviewSamples`, `WriteDraftCasesJSONL`, TODO expected fields, trace ID preservation, deterministic draft ID generation, duplicate suffix handling, `review_required=true`, `expected.release_blocking=false`, unsafe sample rejection, and a loader guard that rejects TODO decisions before promotion.

### GQ5-T03 — Add monthly calibration checklist

- **What:** Added a monthly calibration checklist for thresholds, prompts, models, source drift, metrics, and case promotion.
- **Where:** `docs/planning/AI_QUALITY_EVALOPS.md` and `docs/guides/VERIFICATION.md`.
- **When:** 2026-05-07.
- **Why:** Eval quality can drift as collectors, source sites, models, prompts, thresholds, and alert patterns change.
- **How:** Documented recurring review commands, issue-class review, threshold checks, prompt/model version recording, source drift checks, cost/latency review, and reviewed-case promotion rules, then marked the GQ5 phase gate complete.

---

## 2026-05-07 — GQ4 Runtime telemetry and review sampling

### GQ4-T01 — Code: trace event model

- **What:** Added safe runtime trace event structs for collector, enrichment, LLM, notification, and alert stages.
- **Where:** `internal/evalops/telemetry.go`, `internal/evalops/telemetry_test.go`, and `internal/evalops/report.go`.
- **When:** 2026-05-07.
- **Why:** Production-like quality telemetry needs traceable event payloads that are safe for logs, Loki, Sentry breadcrumbs, and OpenTelemetry attributes.
- **How:** Implemented `TraceStage`, `TraceEvent`, safe attribute mapping, telemetry attribute export, oversized source truncation, and shared redaction for API keys, webhook URLs, emails, phones, tokens, and raw PII.

### GQ4-T02 — Code: metric counters and histograms

- **What:** Added registry-scoped Prometheus metrics for pipeline and AI quality signals.
- **Where:** `internal/evalops/metrics.go` and `internal/evalops/metrics_test.go`.
- **When:** 2026-05-07.
- **Why:** GroupScout needs repeatable collector, enrichment, LLM, alert, latency, token, and cost signals without relying on the global Prometheus registry in tests.
- **How:** Implemented `RuntimeMetrics` counters and histograms with injected registry registration, duplicate-registration reuse, and helpers for collector failures, enrichment skips, LLM errors, alert decisions, stage latency, token counts, and estimated cost cents.

### GQ4-T03 — Code: review sample writer

- **What:** Added a redacted append-only JSONL writer for production review samples.
- **Where:** `internal/evalops/review_sample.go` and `internal/evalops/review_sample_test.go`.
- **When:** 2026-05-07.
- **Why:** Runtime failures need enough safe trace and case metadata to become future eval cases without leaking secrets, PII, or raw sensitive source text.
- **How:** Implemented disabled-by-config behavior, timestamp injection, review-required metadata, JSONL append writes, unsafe attribute rejection, result redaction, and per-field truncation.

### GQ4-T04 — Add dashboard/alert checklist

- **What:** Documented GQ4 telemetry verification and dashboard/alert candidates.
- **Where:** `docs/guides/VERIFICATION.md` and `docs/planning/AI_QUALITY_EVALOPS.md`.
- **When:** 2026-05-07.
- **Why:** Operators need a concrete checklist for using safe telemetry to spot hallucination, drift, cost, collector, and webhook regressions.
- **How:** Added verification commands, trace/metric/sample checks, Grafana/Sentry/Loki alert candidates, and marked the GQ4 tasks plus phase gate complete.

---

## 2026-05-07 — GQ3 Promptfoo target and CI gate

### GQ3-T01 — Code: local eval target HTTP server

- **What:** Added a local HTTP eval target for Promptfoo-compatible offline case scoring.
- **Where:** `internal/evalops/target.go`, `internal/evalops/target_test.go`, and `cmd/evaltarget/main.go`.
- **When:** 2026-05-07.
- **Why:** GQ3 needs Promptfoo to call GroupScout quality checks through Go without JavaScript glue or live services.
- **How:** Implemented `/eval/run` with JSON validation, case lookup, trace ID propagation, request timeouts, injectable executor/scorer interfaces, safe response fields, and deterministic default scoring.

### GQ3-T02 — Add Promptfoo YAML configs

- **What:** Added Promptfoo and threshold configuration for the GroupScout golden fixture set.
- **Where:** `evals/promptfoo/groupscout.yaml` and `evals/promptfoo/thresholds.yaml`.
- **When:** 2026-05-07.
- **Why:** Promptfoo needs a stable config that passes case variables to the local target and avoids live provider keys by default.
- **How:** Configured the HTTP provider to call `http://localhost:18080/eval/run`, pass `case_id` and trace values, extract `json.output`, and enumerate all 22 synthetic cases.

### GQ3-T03 — Code: release gate command

- **What:** Added a release gate that reads eval report JSON and YAML thresholds.
- **Where:** `internal/evalops/gate.go`, `internal/evalops/gate_test.go`, and `cmd/evalgate/main.go`.
- **When:** 2026-05-07.
- **Why:** Critical or release-blocking eval regressions must produce deterministic non-zero exits for CI while warning-only reports stay configurable.
- **How:** Implemented threshold parsing, report loading, summary text, exit-code decisions, warning-as-error behavior, and missing/malformed file errors.

### GQ3-T04 — Add Makefile/CI commands

- **What:** Added offline eval report and gate commands plus documentation for local and CI-safe usage.
- **Where:** `Makefile`, `cmd/evalquality/main.go`, `internal/evalops/quality.go`, `internal/evalops/quality_test.go`, `docs/guides/TESTING.md`, `README.md`, and `docs/planning/AI_QUALITY_EVALOPS.md`.
- **When:** 2026-05-07.
- **Why:** Developers and CI need one-command quality reports and gate enforcement with discoverable instructions.
- **How:** Added `make eval-target`, `make eval-quality`, and `make eval-gate`; wrote JSON, Markdown, and JUnit artifacts under `build/evals`; and marked the GQ3 phase gate complete.

---

## 2026-05-07 — GQ2 Go eval loader and deterministic scorers

### GQ2-T01 — Code: eval case loader

- **What:** Added a typed JSONL case loader with deterministic file/line ordering and aggregated validation errors.
- **Where:** `internal/evalops/cases.go`, `internal/evalops/types.go`, and `internal/evalops/cases_test.go`.
- **When:** 2026-05-07.
- **Why:** GQ2 needs a local, repeatable way to load AI quality fixtures without dereferencing fixture URLs or depending on live services.
- **How:** Implemented strict enum and required-field validation for IDs, case types, source systems/types, risk levels, decisions, expected outcomes, and duplicate IDs across files.

### GQ2-T02 — Code: lead relevance scorer

- **What:** Added a deterministic lead relevance scorer for keep/drop/review decisions, score bands, duplicate raw hashes, and missing source evidence warnings.
- **Where:** `internal/evalops/lead_scorer.go` and `internal/evalops/lead_scorer_test.go`.
- **When:** 2026-05-07.
- **Why:** Lead quality regressions must be caught before collectors, prompt changes, or enrichment rules create unsafe high-priority leads.
- **How:** Implemented fixture-contract rules for commercial permits, residential drops, duplicate revisions, film production, event relevance, malformed input, missing value, and conflicting evidence.

### GQ2-T03 — Code: enrichment completeness scorer

- **What:** Added enrichment completeness checks for room-night estimates, project duration, lodging need, rationale, source evidence, confidence, unknown handling, and contradictory evidence.
- **Where:** `internal/evalops/enrichment_scorer.go` and `internal/evalops/enrichment_scorer_test.go`.
- **When:** 2026-05-07.
- **Why:** AI-enriched leads must stay explicit about uncertainty and fail closed on confident unsupported claims.
- **How:** Implemented range validation, required-field checks, low-confidence unknown labeling, and critical failures for high-confidence contradictions or unsupported estimates.

### GQ2-T04 — Code: outreach safety scorer

- **What:** Added outreach draft safety checks for `draft_for_review`, fabricated claims, aggressive CTAs, PII/secret leakage, and safe no-draft behavior for dropped leads.
- **Where:** `internal/evalops/outreach_scorer.go` and `internal/evalops/outreach_scorer_test.go`.
- **When:** 2026-05-07.
- **Why:** Outreach must never auto-send, leak sensitive source data, or repeat unsupported claims from adversarial fixtures.
- **How:** Implemented deterministic string and regex checks for forbidden fixture claims, redaction requirements, email/phone/token/webhook patterns, urgent CTA language, and source-backed personalization.

### GQ2-T05 — Code: alert threshold scorer

- **What:** Added a deterministic airport disruption alert scorer for delay watches, weather-only noise, stale-data fail-closed behavior, missing NOTAM degradation, threshold boundaries, and priority Slack signal text.
- **Where:** `internal/evalops/alert_scorer.go` and `internal/evalops/alert_scorer_test.go`.
- **When:** 2026-05-07.
- **Why:** YVR disruption alerts must avoid false priority pages from stale data or one weak signal while still escalating true multi-signal disruptions.
- **How:** Implemented a fixture-contract alert score from cancellation, weather, NOTAM, duration, time-of-day, runway, and feed-age inputs, with warning-level degraded output for missing NOTAM data.

### GQ2-T06 — Code: Markdown and JUnit reports

- **What:** Added deterministic eval report generation for JSON summaries, Markdown summaries, and JUnit XML.
- **Where:** `internal/evalops/report.go` and `internal/evalops/report_test.go`.
- **When:** 2026-05-07.
- **Why:** GQ2 results need CI-uploadable artifacts that make critical and warning failures actionable without leaking sensitive fixture content.
- **How:** Implemented severity/case/scorer sorting, summary counts, release-blocking failure counts, XML output, Markdown tables, and report redaction for secret-like tokens, emails, and phone numbers.

### GQ2 documentation hygiene

- **What:** Marked GQ2 tasks complete, documented the evalops test command, and updated fixture/schema wording now that the loader exists.
- **Where:** `docs/planning/AI_QUALITY_EVALOPS.md`, `docs/guides/TESTING.md`, `docs/evals/groupscout-case-schema.md`, `data/evals/groupscout/README.md`, and `docs/CHANGELOG.md`.
- **When:** 2026-05-07.
- **Why:** The planning board, fixture docs, and test guide must reflect the implemented GQ2 behavior.
- **How:** Updated Markdown checkboxes, added `go test -v ./internal/evalops`, retitled the fixture README, and recorded what/where/when/why/how for every GQ2 task.

---

## 2026-05-07 — GQ1 golden lead and alert fixtures

### GQ1-T01 — Create lead eval case schema

- **What:** Added the JSONL schema for GroupScout eval cases, including source metadata, raw source text, expected decisions, score bands, enrichment expectations, evidence requirements, forbidden claims, privacy checks, and loader validation rules.
- **Where:** `docs/evals/groupscout-case-schema.md`.
- **When:** 2026-05-07.
- **Why:** GQ2 needs a stable fixture contract before implementing the Go case loader and deterministic scorers.
- **How:** Documented one-case-per-line JSONL files, required fields, accepted values, lead-specific enrichment fields, alert-specific expectations, and future loader rejection rules.

### GQ1-T02 — Add construction and permit cases

- **What:** Added synthetic construction and permit cases covering a large commercial project, low-value residential renovation, duplicate permit revision, malformed PDF text, and missing declared value.
- **Where:** `data/evals/groupscout/lead_cases.jsonl`.
- **When:** 2026-05-07.
- **Why:** Lead relevance and enrichment scoring need deterministic positive, negative, duplicate, malformed, and unknown-value examples.
- **How:** Added expected keep/drop/review decisions, score bands, room-night ranges, evidence requirements, forbidden claims, and critical failure conditions for each case.

### GQ1-T03 — Add film, event, bid, and infrastructure cases

- **What:** Added synthetic Creative BC, VCC, Eventbrite, BCBid, and infrastructure announcement cases.
- **Where:** `data/evals/groupscout/lead_cases.jsonl`.
- **When:** 2026-05-07.
- **Why:** The golden set must cover the non-permit source mix that GroupScout already collects.
- **How:** Added source-specific raw text, metadata, expected lodging relevance, score bands, evidence anchors, and forbidden unsupported claims.

### GQ1-T04 — Add airport disruption alert cases

- **What:** Added synthetic YVR alert cases for delay spike watch, weather-only noise, NOTAM-only noise, multi-signal hard alert, and stale-data fail-closed behavior.
- **Where:** `data/evals/groupscout/alert_cases.jsonl`.
- **When:** 2026-05-07.
- **Why:** Alert quality gates need deterministic fixtures that distinguish true priority disruptions from weak or stale signals.
- **How:** Added multi-signal snapshots with cancellation, weather, NOTAM, runway, duration, and feed freshness inputs plus expected alert states and critical failure conditions.

### GQ1-T05 — Add adversarial and privacy cases

- **What:** Added synthetic adversarial cases for prompt injection, raw PII, conflicting source dates, suspicious contact data, and secret-like token text.
- **Where:** `data/evals/groupscout/adversarial_cases.jsonl`.
- **When:** 2026-05-07.
- **Why:** Release gates must fail closed for unsafe outreach, unsupported claims, prompt-injection leakage, and privacy regressions.
- **How:** Added privacy redaction requirements, forbidden output patterns, low-confidence review expectations, and critical mismatch examples.

### GQ1 documentation hygiene

- **What:** Marked the GQ1 tasks and phase gate complete, added a fixture README, and linked the schema from the project README.
- **Where:** `docs/planning/AI_QUALITY_EVALOPS.md`, `data/evals/groupscout/README.md`, and `README.md`.
- **When:** 2026-05-07.
- **Why:** The fixture set needs to be discoverable and the planning checklist must reflect the implemented GQ1 scope.
- **How:** Updated Markdown task boxes, documented fixture file counts and rules, and added a README documentation link.

---

## 2026-05-07 — GQ0 AI Quality EvalOps documentation

### GQ0-T01 — Define quality dimensions

- **What:** Added measurable AI quality dimensions for lead relevance, source evidence, enrichment completeness, room-night rationale, outreach safety, alert correctness, latency, and cost.
- **Where:** `docs/planning/AI_QUALITY_EVALOPS.md`; cross-referenced from `docs/prompts/AI_DATA_STRATEGY.md`.
- **When:** 2026-05-07.
- **Why:** GQ0 needs a concrete definition of "good AI output" before fixture, scorer, gate, or telemetry implementation starts.
- **How:** Added a dimension table with measurable signals and critical failure examples, then marked `GQ0-T01` complete.

### GQ0-T02 — Define critical failure gates

- **What:** Documented release-blocking failures for unsupported high-priority claims, fabricated evidence, unsafe outreach, secret/PII leakage, false priority alerts, stale-data priority decisions, and unexpected live calls in deterministic evals.
- **Where:** `docs/planning/AI_QUALITY_EVALOPS.md` and the EvalOps baseline note in `docs/prompts/AI_DATA_STRATEGY.md`.
- **When:** 2026-05-07.
- **Why:** The AI quality gate must fail closed for safety, privacy, unsupported-claim, and alert correctness regressions.
- **How:** Added a critical-gates section and clarified that warning-level findings pass only when explicitly degraded, review-only, or configured as non-blocking.

### GQ0-T03 — Define deterministic vs live eval split

- **What:** Defined deterministic evals as the default release-blocking path and live evals as explicit manual calibration only.
- **Where:** `docs/planning/AI_QUALITY_EVALOPS.md`; reinforced in `docs/prompts/AI_DATA_STRATEGY.md`.
- **When:** 2026-05-07.
- **Why:** Local development and CI should not depend on live websites, LLM providers, Slack, SendGrid, Sentry, Ollama, or other external services.
- **How:** Added deterministic and live eval rules covering fixtures, fake clients, `httptest`, credentials, manual review, and release-blocking behavior.

### GQ0-T04 — Add README documentation links

- **What:** Added README links for the EvalOps plan, AI quality TDD policy, and AI quality implementation prompts.
- **Where:** `README.md`.
- **When:** 2026-05-07.
- **Why:** The AI quality documentation pack needs to be discoverable from the project root before downstream GQ1+ work begins.
- **How:** Extended the README documentation list with `docs/planning/AI_QUALITY_EVALOPS.md`, `docs/guides/TDD_AI_QUALITY.md`, and `docs/prompts/PROMPTS_AI_QUALITY.md`.

### GQ0 documentation hygiene

- **What:** Fixed AI quality prompt links, corrected the AI data strategy reference, and added the EvalOps baseline to the AI data strategy.
- **Where:** `docs/planning/AI_QUALITY_EVALOPS.md` and `docs/prompts/AI_DATA_STRATEGY.md`.
- **When:** 2026-05-07.
- **Why:** Broken handoff links would slow GQ2+ implementation and make the GQ0 task board less reliable.
- **How:** Updated prompt links from `../PROMPTS_AI_QUALITY.md` to `../prompts/PROMPTS_AI_QUALITY.md`, changed the GQ0 file list to link directly to `docs/prompts/AI_DATA_STRATEGY.md`, and documented how observability should feed deterministic quality gates.

---

## LUX Platform Pipelines (feat/mvps branch)

Standalone n8n workflows for LUX Construction. These live in `docs/mvps/` and run entirely in n8n — no GroupScout Go server, Postgres, or Ollama required.

### MVP-B — Automated Lead Follow-Up Sequence

New pipeline: accepts an inbound lead payload, classifies it with Claude, routes to a commercial or residential 3-email sequence generator, logs all drafts to Airtable, and notifies `#new-leads` in Slack.

- `docs/mvps/mvp-b.md` — full spec (prompts, routing logic, Airtable schema, demo script)
- `docs/mvps/mvp-b/` — implementation directory:
  - `payload.json` — commercial lead, high tier (Marcus Webb)
  - `payload_alt.json` — residential lead, medium tier (Jennifer Okafor)
  - `prompts/system_brand_voice.txt` — LUX sales voice (Prompt 1, applied to every call)
  - `prompts/user_classify.txt` — lead classification prompt (Prompt 2, Haiku)
  - `prompts/user_sequence_commercial.txt` — commercial 3-email sequence (Prompt 3A, Sonnet)
  - `prompts/user_sequence_residential.txt` — residential 3-email sequence (Prompt 3B, Sonnet)
  - `prompts/user_slack_notify.txt` — internal Slack notification (Prompt 4, Haiku)
  - `n8n_workflow.json` — 14-node importable workflow
  - `README.md` — pipeline doc, model table, demo script
- `.claude/agents/lux-lead-classifier.md` — classifies leads, enforces tier/category rules
- `.claude/agents/lux-lead-sequence-writer.md` — generates sequences, routes commercial vs residential
- `docs/guides/LUX_MVP_B_SETUP.md` — n8n import, Airtable schema, credential setup, curl test commands
- `docs/guides/LUX_MVP_B_USER_GUIDE.md` — daily usage, email checklist, manual trigger
- `docs/guides/LUX_MVP_B_TROUBLESHOOTING.md` — 10 failure scenarios with root cause and fix

### MVP-A — AI Client Status Email Generator

Pipeline that accepts a JobTread project payload, generates a professional client-facing status email via Claude, and posts the draft to `#client-updates-review` for PM review before anything reaches the client.

- `docs/mvps/mvp-a/` — payloads, prompts, `n8n_workflow.json`, `README.md`
- `.claude/agents/lux-status-email-writer.md` — enforces budget variance rules, milestone filtering, client action item detection
- `docs/guides/LUX_MVP_A_SETUP.md` — setup guide with verify checklists for both test payloads
- `docs/guides/LUX_MVP_A_USER_GUIDE.md` — PM workflow, approval process
- `docs/guides/LUX_MVP_A_TROUBLESHOOTING.md` — debugging guide

### MVP-C — LinkedIn & Podcast Post Pipeline

Pipeline that generates LinkedIn post drafts from project milestone or podcast episode data and delivers them to `#content-review` for human review.

- `docs/mvps/mvp-c/` — payloads, prompts, `n8n_workflow.json`
- `.claude/agents/lux-content-writer.md`, `lux-slack-notifier.md`
- `docs/guides/LUX_MVP_C_SETUP.md` — setup guide with curl test commands for both payload types
- `docs/guides/LUX_MVP_C_USER_GUIDE.md` — content team workflow
- `docs/guides/LUX_MVP_C_TROUBLESHOOTING.md` — debugging guide

### Shared Infrastructure

- `docs/guides/N8N_GUIDE.md` — extended with:
  - § 9: Operations Reference (start/stop, executions, retries, credentials, env vars)
  - § 10: Workflow Inventory (GroupScout + all three MVP workflows)
  - § 11: Slack Integration (Bot Token setup for MVP-A/B, Incoming Webhook for MVP-C/GroupScout, error table, channel ID guidance)
- `docs/guides/TESTING.md` — extended with § 11: LUX MVP Testing (n8n only — `docker compose up -d n8n`, curl commands, verify checklists for all three MVPs, test mode instructions)

---

## v0.1.0

- ✅ Phase 13.1: Implemented utilityBase, fetchResult, and helpers
  - Utility region filtering and HTTP fetching infrastructure
  - Hashing, currency parsing, and date formatting utilities
  - 100% test coverage for base functionality

Co-Authored-By: qwen <noreply@example.com>
