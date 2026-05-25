# ROADMAP.md — groupscout

> **Master roadmap** consolidating `PHASES.md`, `AI.md`, `FUTURE_INTEGRATION.md`, `TECH_IDEAS.md`, and `PLANNING.md` into one strategic view.
>
> - Live work is tracked in Beads. `PHASES.md` remains the **historical phase reference** (checkboxes + acceptance context).
> - This file is the **big-picture list** — what's done, what's next, what's future.
> - Source docs are preserved as-is; this is a read-only synthesis for navigation.

---

## Live Beads Follow-Ups

Roadmap checkboxes are historical/strategic context. The missed-task audit on 2026-05-25 filed the concrete follow-up work in Beads:

- `groupscout-site-fuc` — guaranteed Sunday/Wednesday one-lead backend delivery.
- `groupscout-site-yfl` — export and smoke-test the Sunday/Wednesday n8n workflow asset. Asset added under `backend/docs/workflows/n8n/`.
- `groupscout-site-j73` — raw audit retention purge worker.
- `groupscout-site-mt5` and `groupscout-site-e5a` — production UI Compose wiring and backend plus UI Docker E2E gate.
- `groupscout-site-3gq` — shared `alertd` state for `/api/alerts`.
- `groupscout-site-yyj` — ops metrics and collector freshness observability.
- `groupscout-site-vud` and `groupscout-site-48g` — LLM provider abstraction plus AI-ready SQL/LLM observability runtime.
- `groupscout-site-b25` — event-driven `POST /ingest` and `EnrichOne`.
- `groupscout-site-39g` — home-deploy runbook execution and restore smoke. Coolify guide was added in `backend/docs/guides/COOLIFY.md`.
- `groupscout-site-29q`, `groupscout-site-4cv`, and `groupscout-site-dxq` — generated UI API client, sanitized raw preview, and UI roadmap schema cleanup.
- `groupscout-site-aaj`, `groupscout-site-1ee`, and `groupscout-site-a1h` — next source-expansion slice, multi-property design, and Codex plugin skill drift.

---

## Phase Summary

- [x] **Phase 1** — Foundation: DB boots, schema applied
- [x] **Phase 2** — Richmond → Claude → Slack (first full pipeline)
- [x] **Phase 3** — Dedup hardened, BC Bid/Delta added, n8n trigger
- [x] **Phase 4** — Creative BC, VCC, Eventbrite, news, announcements, instant alert, email digest ✅
- [ ] **Phase 5** — Smart refresh: avoid redundant PDF fetches *(deferred)*
- [x] **Phase 6** — Productionize: Docker, Postgres, VPS deploy ✅
- [ ] **Phase 7** — User requests & API refinements *(in progress)*
- [ ] **Phase 8** — System reliability & observability *(in progress; `groupscout-site-yyj`)*
- [ ] **Phase 9** — Architecture & scaling: concurrency, caching
- [ ] **Phase 10** — Ecosystem & UI: dashboard, CRM, extensions (`groupscout-site-mt5`, `groupscout-site-e5a`, `groupscout-site-3gq`, `groupscout-site-29q`, `groupscout-site-4cv`, `groupscout-site-dxq`, `groupscout-site-kb4`)
- [ ] **Phase 11** — CRM direct integration: HubSpot, Salesforce
- [ ] **Phase 12** — Source expansion: Metro Vancouver municipalities (`groupscout-site-aaj` selects the next slice)
- [ ] **Phase 13** — Public tenders & utilities: BC Hydro, FortisBC (`groupscout-site-aaj` selects the next slice)
- [ ] **Phase 14** — Infrastructure & self-hosting: Docker ecosystem *(in progress; `groupscout-site-yfl`, `groupscout-site-fuc`)*
- [x] **Phase 15** — PostgreSQL + pgvector migration: production storage + RAG foundation ✅
- [ ] **Phase 16** — LLM Provider Abstraction: no vendor lock-in (Claude / OpenAI / Ollama / Gemini) 🔄 (`groupscout-site-vud`)
- [x] **Phase 17** — Airport Disruption Alert System (`alertd`): YVR real-time monitoring ✅
- [ ] **Phase 18** — Contact Enrichment: Hunter.io integration & budget tiers 📋
- [ ] **Phase 19** — Slack Actions & Lead Feedback: claim/dismiss/snooze buttons 📋
- [x] **Phase 20** — Housekeeping & Developer Experience ✅ (`groupscout-site-a1h`, `groupscout-site-2h1`)
- [x] **Phase 21** — Ollama Prod Hardening ✅
- [ ] **Phase 22** — Multi-Property Support: portfolio routing & YAML config 📋 (`groupscout-site-1ee`)
- [ ] **Phase 23** — Advanced Intelligence: Repeat Detection & Signal Quality 📋
- [ ] **Phase 24** — Agentic Reasoning & Tool-Calling: ReAct loop + BC Registry / LinkedIn tools 📋
- [ ] **Phase 25** — AI Observability & Quality: Langfuse + AI-Ready SQL + eval harness 📋 (`groupscout-site-48g`)
- [ ] **Phase 26** — Production Deployment: Hetzner + Coolify (primary), Railway (managed alt), event-driven `/ingest` endpoint 📋 (`groupscout-site-39g`, `groupscout-site-b25`; Coolify guide complete)
- [ ] **Phase 27** — Input Audit & Verification Trail ◐ raw audit/redaction complete; retention cleanup remains open (`groupscout-site-j73`; sanitized preview: `groupscout-site-4cv`)
- [ ] **Phase 28** — Analytics & Source Attribution 📋
- [ ] **Phase 29** — Prompt Engineering & Strict TDD 📋
- [ ] **Phase 30** — Advanced Audit & Verification 📋

---


## Phase 4 — More Sources + Full Notifications ✅

### Collectors
- [x] Google News RSS monitor (`internal/collector/news.go`)
- [x] Creative BC "In Production" scraper (`internal/collector/creativebc.go`)
- [x] Vancouver Convention Centre event scraper (`internal/collector/vcc.go`)
- [x] Eventbrite conference scraper (`internal/collector/eventbrite.go`)
- [x] Infrastructure announcements — BCIB, TransLink, YVR (`internal/collector/announcements.go`)

### Notifications
- [x] Instant alert for priority score ≥ 9 (`internal/enrichment/enricher.go`)
- [x] Weekly email digest via Resend (`internal/leadnotify/email.go`)
- [x] Claude outreach draft generation (`internal/enrichment/claude.go`)
- [x] `/digest` HTTP endpoint for n8n triggers

---

## Phase 5 — Smart Refresh ⏸ (Deferred)

> Richmond publishes weekly; a weekly cron is sufficient for now.
> Add this when reducing unnecessary PDF downloads matters.

- [ ] `migrations/002_scrape_state.up.sql` — add `scrape_state` table
- [ ] `internal/storage/scrapestate.go` — `ScrapeStateStore` interface + SQLite impl
- [ ] `internal/collector/richmond.go` — hash reports page HTML before downloading PDFs
- [ ] `internal/collector/richmond.go` — diff current vs. previously seen PDF links; download new only
- [ ] Skip entire run if page hash unchanged

---

## Phase 6 — Productionize ✅

- [x] Dockerfile + docker-compose for app + Postgres
- [x] Postgres migration path (`DATABASE_URL=postgres://...`)
- [x] `golang-migrate/migrate` wired to migration files
- [x] VPS or Railway deployment (single container)
- [x] Env var hardening + `.env.example` documentation
- [x] Smoke test all collectors end-to-end on production
- [x] Add PostgreSQL support with SQLite fallback
- [x] Add versioned migrations for Postgres schema
- [x] Add `pgvector` support and SQLite vector storage fallback

---

## Phase 7 — User Requests & API Refinements 🔄

- [x] Slack notifications show lead source
- [ ] `internal/storage/leads.go` — `List` with filtering (status, score) and pagination
- [ ] `cmd/server/main.go` — `GET /leads` endpoint (filter by status, source, min_score)
- [ ] `cmd/server/main.go` — `PATCH /leads/{id}` endpoint (update status, add notes)
- [ ] `cmd/server/main.go` — `GET /leads/{id}` endpoint (include outreach history)
- [ ] `cmd/server/main.go` — `POST /leads/{id}/outreach` endpoint (log outreach attempt)
- [ ] `internal/storage/outreach.go` — `OutreachLogStore` for tracking interactions

---

## Phase 8 — System Reliability & Observability 🔄

- [x] Structured logging via `log/slog`
- [x] Sentry integration — runtime errors + pipeline failures
- [x] `/health` endpoint — DB ping + critical env var check
- [x] Pipeline run metrics (counts logged)
- [x] Basic lead deduplication (hash-based)
- [x] `TESTING.md` created
- [ ] Distributed tracing via OpenTelemetry — visualize lead flow end-to-end
- [ ] Prometheus `/metrics` endpoint — counters for leads collected, enriched, notified
- [ ] Grafana Loki dashboard — structured log aggregation
- [ ] Advanced health check — last successful run time per collector

Tracked follow-up: `groupscout-site-yyj` owns Prometheus metrics, collector freshness in health/system data, and dashboard documentation.

---

## Phase 9 — Architecture & Scaling 📋

- [ ] Collector registry — centralized registry to simplify adding/removing sources
- [ ] Parallel collector execution — goroutine worker pools (reduces total pipeline runtime)
- [ ] Dynamic run parameters — pass keyword/date overrides via `/run` endpoint
- [ ] Redis caching layer — cache Claude enrichment results + scraper states
- [ ] Incremental scraping — only process data newer than last successful run
- [ ] `GET /collectors` — list active collectors and last run status
- [ ] `POST /collectors/{name}/run` — manually trigger a specific collector
- [ ] `GET /stats` — aggregated metrics (total leads, enrichment rate, score distribution)

---

## Phase 10 — Ecosystem & UI 📋

> Product and integration strategy: [UI_STRATEGY.md](./ui/UI_STRATEGY.md).
> Backend-owned UI API contracts and smoke paths are now tracked in [UI_TDD_PHASE_PLAN.md](./ui/UI_TDD_PHASE_PLAN.md) Phases 31-38. The broad checklist below is retained as strategic product scope rather than authoritative implementation status.
> UI design adaptation: [UI_DESIGN_SYSTEM.md](./ui/UI_DESIGN_SYSTEM.md).
> UI shell/routing contract: [UI_PHASE32_APP_SHELL_ROUTING_CONTRACT.md](./ui/UI_PHASE32_APP_SHELL_ROUTING_CONTRACT.md).
> UI mocked lead inbox contract: [UI_PHASE33_MOCKED_LEAD_INBOX_CONTRACT.md](./ui/UI_PHASE33_MOCKED_LEAD_INBOX_CONTRACT.md).
> UI lead detail/evidence contract: [UI_PHASE34_LEAD_DETAIL_EVIDENCE_CONTRACT.md](./ui/UI_PHASE34_LEAD_DETAIL_EVIDENCE_CONTRACT.md).
> UI endpoint contracts: [UI_API_ENDPOINTS.md](./ui/UI_API_ENDPOINTS.md).
> Strict TDD phase plan: [UI_TDD_PHASE_PLAN.md](./ui/UI_TDD_PHASE_PLAN.md).
> Copy-paste prompts: [PROMPTS_PHASE31_UI.md](../prompts/PROMPTS_PHASE31_UI.md), [PROMPTS_PHASE32_UI.md](../prompts/PROMPTS_PHASE32_UI.md), [PROMPTS_PHASE33_UI.md](../prompts/PROMPTS_PHASE33_UI.md), and [PROMPTS_PHASE34_UI.md](../prompts/PROMPTS_PHASE34_UI.md).

- [ ] Operator UI foundation — TypeScript React SPA under a future `web/` app, backed by `/api/*` contracts and same-origin deployment
- [ ] Lead inbox — filterable triage table by status, score, source, property, owner, and timing
- [ ] Lead detail — source evidence, raw audit access, AI rationale, notes, status actions, and outreach history
- [ ] Verification queue — flagged or low-confidence leads with reviewer corrections and audit trail
- [ ] Pipeline monitor — collector run health, failures, LLM latency/errors, and webhook delivery status
- [ ] Analytics — source attribution, claimed/won/lost rate, score distribution, aging leads, and demand calendar
- [ ] CRM sync — one-click push to HubSpot/Salesforce/Pipedrive via their APIs
- [ ] Multi-persona outreach drafting — A/B templates (e.g., "Technical Specialist" vs. "Sales Executive")
- [ ] Chrome Extension — manually clip a lead from any webpage into the pipeline
- [ ] Smart scheduling — scrapers run more frequently during high-activity periods
- [ ] Follow-up reminders based on lead status; keep auto-sending outreach out of the MVP

---

## Phase 11 — CRM Integration 📋

- [ ] HubSpot sync — auto-push leads with score > 8 to HubSpot via API
- [ ] LinkedIn helper — API endpoint to generate pre-filled LinkedIn search URLs for GCs/applicants
- [ ] Salesforce connector — create/update contact + opportunity records
- [ ] "One-Click Outreach" — button in Slack notification → triggers email draft + CRM record creation

---

## Phase 12 — Source Expansion (Metro Vancouver) 📋

> Collector interface is additive — no core pipeline changes needed.

### Municipal Building Permits
- [ ] **Burnaby** — daily/weekly PDF reports
- [ ] **Vancouver** — Open Data Portal (CSV/API)
- [ ] **Coquitlam** — monthly PDF reports
- [ ] **Surrey** — monthly building permit summary PDFs
- [ ] **New Westminster** — weekly building permit reports
- [ ] **North Vancouver (DNV)** — monthly building permit reports

### Specialized Industry Sources
- [ ] **Journal of Commerce (Canada)** — major BC project starts + contract awards; priority on Richmond/Delta
- [ ] **Daily Hive Vancouver / Vancouver Sun** — "hotel pipeline", "office-to-hotel conversion" keywords
- [ ] **PNE / Playland** — large trade shows or seasonal events
- [ ] **Abbotsford Centre / Langley Events Centre** — regional events

### Future Lead Segments
- [ ] Sports teams — Rogers Arena schedule, BC Lions, Vancouver FC fixtures
- [ ] Film / TV crews — BC film permit registry
- [ ] Government contractors — DND, CBSA, Transport Canada near YVR
- [ ] Touring acts — venue booking announcements

---

## Phase 13 — Public Tenders & Utilities 📋

- [ ] BC Hydro — major utility infrastructure project announcements + tenders
- [ ] FortisBC — major utility projects (often 2-3 year crew deployments)

---

## Phase 14 — Infrastructure & Self-Hosting 🔄

> Full tool options (Docker vs cloud) for embeddings, vector stores, and observability: see [AI_DATA_STRATEGY.md](../prompts/AI_DATA_STRATEGY.md).

**Automation (existing):**
- [x] Docker Compose — GroupScout + n8n + Redis
- [x] n8n workflow — trigger `/run` and `/digest` on schedule
- [x] `/n8n/webhook` endpoint — receive external leads from n8n

**Monitoring (existing):**
- [x] Prometheus + Grafana Loki — infrastructure monitoring + log aggregation

**Analytics:**
- [ ] Metabase or Grafana — connect to `groupscout.db` for lead analytics dashboard

**LLM Observability:**
- [ ] Langfuse — track Claude API calls, token costs, prompt versions, hallucinations
  - Cloud: free hobby tier (start here)
  - Docker: `langfuse/langfuse` + Postgres backend (add at Phase 6)

**Embeddings (for RAG):**
- [ ] Ollama (`ollama/ollama`) — local embedding server (`nomic-embed-text`); free, no external API
  - Cloud alternative: Voyage AI `voyage-3-lite` (free 200M tokens/month, best Claude pairing)

**Vector Store (for RAG):**
- [ ] Qdrant (`qdrant/qdrant`) — REST API, Go client, lightweight; replaces in-memory cosine at scale
  - Postgres path: pgvector extension instead (no extra container; preferred if on Phase 6 Postgres)

**Search:**
- [ ] Meilisearch — fast full-text lead search for Admin UI

**Notifications:**
- [ ] Gotify or Apprise — alternative/secondary notification channels

**Error Tracking:**
- [ ] Sentry self-hosted (Docker) — if data privacy becomes a concern

---


---

## Phase 15 — PostgreSQL + pgvector Migration ✅

> **Why now:** repository pattern already in place; Docker already running; data is tiny; pgvector unlocks RAG.
> **Replaces** the Postgres portion of Phase 6.
> Full atomic tasks: see `PHASES.md` Phase 15.

**Part A — Postgres container + driver:**
- [x] `docker-compose.yml` — add `pgvector/pgvector:pg17` + named volume + health check
- [x] `go.mod` — add `github.com/jackc/pgx/v5` (pure Go, no CGO); keep SQLite for local dev fallback
- [x] `config/config.go` — `DriverName()` helper; selects driver from `DATABASE_URL` prefix
- [x] `internal/storage/db.go` — `Open()` routes to pgx or SQLite based on driver
- [x] Verify: connects to Postgres; no migrations yet

**Part B — Schema (Postgres-compatible migrations):**
- [x] `migrations/001_init.postgres.up.sql` — native types: `UUID DEFAULT gen_random_uuid()`, `BOOLEAN`, `TIMESTAMPTZ`, `JSONB`
- [x] `internal/storage/db.go` — wire `golang-migrate/migrate` for both drivers
- [x] Verify: migrations run clean; tests pass

**Part C — Storage layer fixes:**
- [x] `internal/storage/leads.go` — remove `boolToInt()`; `?` → `$N` placeholders; native `bool` + `time.Time`
- [x] `internal/storage/raw.go` — `?` → `$N`; `raw_data` writes as JSONB
- [x] Verify: full pipeline end-to-end against Postgres

**Part D — pgvector:**
- [x] `migrations/003_pgvector.up.sql` — `CREATE EXTENSION vector`; `lead_embeddings` with `vector(512)` + ivfflat index
- [x] `internal/storage/embeddings.go` — `PostgresEmbeddingStore` using `<=>` cosine distance
- [x] `internal/enrichment/embeddings.go` — factory: `postgres://` → pgvector; `*.db` → Go cosine
- [x] Verify: similarity search uses index (`EXPLAIN`)

**Part E — Data migration + productionize:**
- [x] `scripts/migrate_to_postgres/main.go` — copy SQLite rows to Postgres in batches
- [x] `.env.example` — update `DATABASE_URL` to Postgres format
- [x] `docker-compose.yml` — app `depends_on` Postgres health check
- [x] `docs/SETUP.md` — update setup instructions
- [x] Verify: `docker compose up` → migrations → full pipeline on Postgres

---

## Phase 16 — LLM Provider Abstraction 🔭

> Remove vendor lock-in from `internal/enrichment`. All LLM calls go through a `LLMClient` interface. Provider is config-driven — switch between Claude, OpenAI, Azure OpenAI, Groq, Mistral, Ollama, or Gemini without touching pipeline code.
> This is separate from the implemented `internal/ollama` operational runtime documented in [OLLAMA_INTEGRATION.md](./OLLAMA_INTEGRATION.md) and [OLLAMA_SETUP.md](../guides/OLLAMA_SETUP.md). Phase 16 is about routing enrichment through a provider abstraction such as `LLM_PROVIDER=ollama`.

- [ ] **Part A** — Interface Extraction: refactor `ClaudeEnricher` to `ClaudeClient` implementing `LLMClient`.
- [ ] **Part B** — OpenAI-Compatible Client: covers OpenAI, Groq, and Mistral.
- [ ] **Part C** — Azure OpenAI Support.
- [ ] **Part D** — Ollama Support: local / Docker, free and private.
- [ ] **Part E** — Fallback & Resilience: primary → secondary on error.

---

## Phase 17 — Airport Disruption Alert System (`alertd`) ✅

**Goal:** A separate real-time binary (`cmd/alertd/`) that monitors YVR flight disruptions and alerts the hotel team via Slack.

### Part A — Data Acquisition & Weather
- [x] ECCC Weather Poller (`internal/weather/eccc.go`)
- [x] YVR Flight Status Scraper (`internal/aviation/yvr.go`)
- [x] NavCanada NOTAM Parser (`internal/aviation/navcanada.go`)

### Part B — Stranded Passenger Score (SPS)
- [x] SPS formula + Vancouver tuning (`internal/aviation/scorer.go`)

### Part C — Alert Lifecycle State Machine
- [x] State machine: Watch → Alert → Update → Resolve (`internal/alert/lifecycle.go`)
- [x] Slack notification with Block Kit (`internal/alert/slack.go`)

### Part D — Hotel Config & Binary
- [x] Hotel ↔ airport config loader (`config/airports.go`)
- [x] Main poll loop (`cmd/alertd/main.go`)

### Part E — Inventory Slash Command
- [x] `/inventory` Slack command handler

---

## Phase 18 — Contact Enrichment 📋

**Goal:** Auto-surface a decision-maker contact alongside each lead (Phase 18).

---

## Phase 19 — Slack Actions & Lead Feedback 📋

> Slack remains the interrupt channel while the web UI becomes the durable review workspace.

- [ ] Slack action buttons — Claim, Dismiss, Snooze
- [ ] Secure Slack action handler — signature verification and status transitions
- [ ] Outreach outcome buttons — Won, Lost, No Response after a lead is claimed
- [ ] Lead aging — resurface snoozed or stale claimed leads
- [ ] Status model alignment with [UI_STRATEGY.md](./ui/UI_STRATEGY.md)

---


---

## Phase 20 — Housekeeping & Developer Experience ✅

**Goal:** Improve local development workflow and project documentation.

- [x] **20.1** `Makefile` — Central hub for build, test, lint, and Docker tasks.
- [x] **20.2** `scripts/test-ollama.sh` — Automated verification of local Ollama setup.
- [x] **20.3** `docs/guides/TESTING.md` — Comprehensive testing guide.
- [x] **20.4** `DEVELOPER.md` — Refactored Makefile-driven workflow.
- [x] **20.5** `.env.example` — Verified essential variables.

---

## Phase 21 — Ollama Prod Hardening ✅

**Goal:** Secure and observe the local LLM infrastructure in production.

- [x] **21.1** `docker-compose.yml` — Added `promtail` for log aggregation.
- [x] **21.2** `config/promtail.yaml` — Configured container log scraping for Loki.
- [x] **21.3** `scripts/backup-volumes.sh` — Automated backup script for all Docker volumes.
- [x] **21.4** `docs/guides/DOCKER.md` — Model management, backup, and monitoring instructions.

---

## Phase 22 — Multi-Property Support 📋

> Configure and run GroupScout for multiple hotel properties with different geographies, segments, and thresholds. No changes to the collector/enrichment core — all config-driven. See `PHASES.md` Phase 22 for atomic tasks.
> Beads follow-up: `groupscout-site-1ee`.

- [ ] **Part A** — Property Config Model: name, location (lat/lng), segments, Slack webhook.
- [ ] **Part B** — Property-Scoped Pipeline: geo-match raw projects against property radius.
- [ ] **Part C** — Alertd Multi-Hotel Config: unify airport alertd and lead property config.

---

## Phase 23 — Advanced Intelligence: Repeat Detection & Signal Quality 📋

> Detect organizations that have brought groups to the market before. Surface signal quality metrics that help the team focus on leads most likely to convert. See `PHASES.md` Phase 23 for atomic tasks.

- [ ] **Part A** — Repeat Pattern Detection: fuzzy-match incoming GCs against historical outreach log wins.
- [ ] **Part B** — Signal Quality Metrics: rate each source by conversion rate (claimed leads / total leads).

---

## Phase 24 — Agentic Reasoning & Tool-Calling 📋

> Upgrade enrichment from single-call LLM to multi-step reasoning for borderline permits (score 5–7, value > $2M). Claude tool-use for BC Registry lookup + LinkedIn URL generation. See `PHASES.md` Phase 24 for atomic tasks.

- [ ] **Part A** — ReAct loop: second LLM call with tool results injected; activated by `REACT_ENABLED=true`
- [ ] **Part B** — Tool implementations: `internal/tools/bc_registry.go` (OrgBook API, no key) + `internal/tools/linkedin.go` (URL builder, no API)
- [ ] **Part C** — Multimodal PDF enrichment via Claude vision (deferred until 5+ cities)

---

## Phase 25 — AI Observability & Quality 📋

> Track LLM call quality, token costs, and prompt versions. Add AI-Ready SQL view to simplify prompt construction. Structural eval harness to catch bad enrichment outputs before they hit Slack. See `PHASES.md` Phase 25 for atomic tasks.
> Beads follow-up: `groupscout-site-48g`.

- [ ] **Part A** — AI-Ready SQL: `v_lead_context` view + `GetContext()` method; refactor prompt builders
- [ ] **Part B** — Langfuse integration: trace per LLM call (model, tokens, latency, score); noop client when key absent
- [ ] **Part C** — Enrichment eval: `EvalLead()` structural validation + Sentry on hard failures

---

## Phase 26 — Production Deployment & Event-Driven Ingestion 📋

> Deploy the full stack to production. **Hetzner CX32 + Coolify (~$10/month) is the primary recommendation** — it runs both binaries as persistent containers, deploys your existing `docker-compose.yml` directly, and includes n8n/Prometheus/Loki at no extra cost. Railway is the managed-PaaS alternative (~$10–18/month). GCP Cloud Run is **incompatible with `alertd`** (scales to zero; daemon needs persistent compute). See `docs/planning/DEPLOYMENT_OPTIONS.md` for the full analysis.
>
> See `PHASES.md` Phase 26 for atomic tasks.
> Beads follow-ups: `groupscout-site-39g` for home deploy and `groupscout-site-b25` for event-driven ingestion. Coolify guidance now lives in `docs/guides/COOLIFY.md`.

- [ ] **Part A** — Home server: No-IP/DuckDNS DDNS + port forwarding → Traefik + Let's Encrypt → optional Cloudflare Tunnel; `docs/guides/HOME_DEPLOY.md` (`groupscout-site-39g`)
- [ ] **Part B** — Hetzner + Coolify cloud deploy (if uptime SLA matters): VPS provision, Coolify setup, backup config; guide complete in `docs/guides/COOLIFY.md`
- [ ] **Part C** — Event-driven ingestion: `POST /ingest` + `EnrichOne()` (platform-agnostic) (`groupscout-site-b25`)
- [ ] **Part D** — Terraform IaC for GCP (optional; alertd needs Compute Engine VM — Cloud Run won't work)

---

## Phase 27 — Input Audit & Verification Trail ◐

**Goal:** Implement a system to store and track all raw inputs (PDFs, API responses, etc.) to allow for verification of the lead enrichment and scoring process.

- [x] **Part A** — Storage Architecture: `raw_inputs` table + `AuditStore` interface
- [x] **Part B** — Collector Integration: save raw PDF/API content before parsing
- [x] **Part C** — Verification Tools: `GET /leads/{id}/raw` + CLI audit tool
- [ ] **Part D** — Retention & Privacy: cleanup worker remains open in [PHASES.md](./PHASES.md) and `groupscout-site-j73`; raw audit redaction policy is documented separately and complete.

---

## Phase 28 — Analytics & Source Attribution 📋

**Goal:** Weekly digest that shows which signal sources are generating closed business, enabling the sales team to prioritize outreach.

- [ ] **Part A** — Source Attribution: Aggregate leads by source + outcome
- [ ] **Part B** — Slack Analytics: Hit Rate % columns in digest
- [ ] **Part C** — Market Demand: Forward demand density view

---

## Phase 29 — Prompt Engineering & Strict TDD 📋

**Goal:** Transition prompts into a formal library with evaluation metrics and strict Test-Driven Development.

- [ ] **Part A — Infrastructure:** Extract hardcoded strings to `assets/prompts/*.tmpl`; implement template loader in `internal/enrichment`.
- [ ] **Part B — Strict TDD:** Create `internal/enrichment/prompts_test.go` with gold standard fixtures.
- [ ] **Part C — Refinement:** Add "Few-Shot" examples for high-priority/complex lead types.

---

## Phase 30 — Advanced Audit & Verification 📋

**Goal:** Transform the raw audit trail into a proactive verification and quality assurance system.

- [ ] **Part A — Verification Workflow:** `POST /leads/{id}/verify` + corrections (JSONB)
- [ ] **Part B — AI Verification Agent:** `Verifier` struct + verification prompt
- [ ] **Part C — Health & Analytics:** Extraction accuracy aggregator + drift detection

---

*groupscout — group lodging demand intelligence*
