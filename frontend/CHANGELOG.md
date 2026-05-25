# Changelog

## Unreleased

### Documentation Consolidation

- What: Consolidated duplicate agent and phase prompt documentation in the centralized frontend docs.
- Where: Updated `CLAUDE.md`, `UI_TDD_PHASE_PROMPTS.md`, `docs/phase-13-product-renderer-runtime.md`, and `docs/ui-tdd-phase-0-15-implementation.md`; removed the retired supporting phase prompt file after merging its useful parallel-agent starters into the canonical prompt archive.
- When: Updated on 2026-05-25 during coordinator task `groupscout-site-fjy`.
- Why: The frontend docs had duplicate Beads instructions, two Phase 0-15 prompt records, and repeated Phase 13 parallel-agent prompts.
- How: Kept `AGENTS.md` canonical for agent workflow, kept `UI_TDD_PHASE_PROMPTS.md` canonical for prompts, kept the Phase 13 prompt pack as the detailed history, and shortened the Phase 13 runtime record to implementation evidence.

### Admin Token Flow Documentation

- What: Added a dedicated admin-token flow doc and expanded UI docs for setup-token login, `groupscout_session`, logout, token rotation, Docker smoke, and stale browser assets.
- Where: Added `docs/admin-token-flow.md`; updated `README.md`, `docs/developer-guide.md`, `docs/how-to-run-backend.md`, `docs/nice-to-knows.md`, `docs/phase-9-session-auth-wrapper-same-origin-deployment.md`, `docs/testing.md`, `docs/troubleshooting.md`, and related backend centralized docs.
- When: Updated on 2026-05-09 during coordinator task `groupscout-site-0o7`.
- Why: The login implementation works, but the operating flow needed to be explicit for testers: read the current setup token, submit it, expect rotation, use logout, and hard-refresh if old immutable assets are cached.
- How: Documented the browser path, direct curl smoke, Docker setup-token command, file-backed versus env-backed token behavior, session TTL/restart caveats, and `admin-login-1` asset version.

### Docker Admin Login Smoke Fix

- What: Fixed the UI Docker runtime so the current admin-login bundle is visible after markdown docs moved out of the UI source repo, and kept the containerized test image useful without mounting the coordinator docs.
- Where: Updated the UI source repo `Dockerfile`, static asset version references, generated `web/dist` assets, and Docker/runtime tests; updated centralized testing and troubleshooting docs.
- When: Updated on 2026-05-09 during coordinator task `groupscout-site-6nh`.
- Why: The running Docker UI did not show login because the containers were old, the Dockerfile still copied missing `DESIGN.md` and `docs/` paths, and browsers could keep the old immutable `pipeline-output-4` JS asset.
- How: Removed stale Markdown copies from the UI Dockerfile, made centralized-doc-only tests skip when docs are not mounted in the isolated container, bumped the static asset query to `admin-login-1`, rebuilt/restarted backend and UI containers, and verified `/api/auth/status` returns `auth_required:true` with the UI serving the new asset version.

### Docker Testing Prep Documentation

- What: Documented the current testing-ready Docker state for manual UI/API smoke, Postgres integration, backend health checks, and the Ollama health caveat.
- Where: Updated `docs/testing.md`, `docs/how-to-run-backend.md`, `docs/developer-guide.md`, `docs/troubleshooting.md`, `docs/nice-to-knows.md`, and related backend centralized docs.
- When: Updated on 2026-05-09 during coordinator task `groupscout-site-1fr`.
- Why: Manual testing needs to know which containers are already spun up, which smoke checks should pass now, and why an Ollama-unavailable health field blocks LLM/enrichment testing but not ordinary UI/API checks.
- How: Confirmed `groupscout_postgres`, `groupscout_app`, `groupscout-groupscout-ui-1`, `groupscout_n8n`, and `groupscout_ollama` status, ran backend/UI smoke curls, and added current commands and caveats to the testing, how-to, developer, troubleshooting, and nice-to-know docs.

### Admin Login Session Completion

- What: Completed the browser-side admin setup-token login flow with protected-route auth checks, post-login session verification, logout API support, a topbar logout control, production startup readiness enforcement, and tests that read centralized markdown docs after docs moved out of the UI source repo.
- Where: Updated `web/src/api/auth.js`, `web/src/app/shell.js`, `web/src/renderer/staticAppEntry.js`, `web/src/renderer/domRenderer.js`, `web/src/renderer/staticStyles.css`, `web/src/server/productionServer.js`, `web/src/server/uiDeployment.js`, generated `web/dist` assets, `test/api-boundary.test.js`, `test/app-shell.test.js`, `test/design-tokens.test.js`, `test/dockerization-contract.test.js`, `test/phase-13-renderer-runtime.test.js`, and `test/session-deployment.test.js`.
- When: Completed on 2026-05-09 during coordinator task `groupscout-site-9ge`.
- Why: The prior `/admin/login` route could submit a setup token, but protected app routes still rendered before session checks, there was no logout control, successful login did not verify the resulting session, production startup did not enforce deployment readiness, and UI tests still assumed markdown docs lived inside the UI source repo.
- How: Added `logout()` to the auth API client, added route-level auth-status checks with unauthenticated redirects to `/admin/login`, verified `/api/auth/status` or `/api/auth/me` after login, rendered a logout button for protected routes, forwarded `/api/auth/logout` before proxy session gating, called deployment readiness before starting the production server, regenerated static assets, and reran focused plus full UI tests.

### Renderer Runtime Review Fixes

- What: Fixed review findings in the dependency-free browser runtime by limiting SPA click interception to app routes, preserving normal `/api/*` raw audit link navigation, serving copied `/src/*` static modules with `no-store`, rendering mobile Verification Queue cards from mobile model data, and removing `.idea` line-ending churn from the worktree.
- Where: Updated `web/src/renderer/staticAppEntry.js`, `web/src/server/productionServer.js`, `web/src/renderer/domRenderer.js`, `test/phase-13-renderer-runtime.test.js`, `test/dockerization-contract.test.js`, `test/verification-queue.test.js`, `docs/phase-13-product-renderer-runtime.md`, `docs/phase-5-verification-queue-raw-audit-review.md`, `docs/testing.md`, and `docs/developer-guide.md`.
- When: Completed on 2026-05-09 during beads task `groupscout-ui-b4l`.
- Why: Raw audit links must reach the same-origin API proxy instead of being swallowed by client-side routing, unbundled copied source modules must not remain stale across deploys, and mobile verification rendering must show the same actionable queue data as the screen model.
- How: Ran parallel worker agents for static-runtime and mobile-verification fixes, added focused renderer/runtime and Docker contract assertions, regenerated static assets, and reran targeted plus full quality gates.

### Phase Prompt Completion Verification

- What: Rechecked the retired supporting Phase 0-15 prompt checklist in sequential order, confirmed the implemented UI baseline still covers every phase through tests and runtime/docs artifacts, closed the production `/api/*` session-gate gap, strengthened deterministic Phase 15 route evidence, corrected Phase 1 token drift, and marked the older supporting checklist as complete.
- Where: Updated the retired supporting phase prompt checklist, `docs/ui-tdd-phase-0-15-implementation.md`, `docs/phase-9-session-auth-wrapper-same-origin-deployment.md`, `docs/phase-15-browser-ux-hardening.md`, `docs/docker-runtime-matrix.md`, `docs/testing.md`, `README.md`, `test/session-deployment.test.js`, `test/browser-ux-hardening.test.js`, `test/app-shell.test.js`, `test/design-tokens.test.js`, `web/src/server/productionServer.js`, `web/src/renderer/domRenderer.js`, `web/src/renderer/browserUxHardening.js`, `web/src/app/shell.js`, and `web/src/design/tokens.js`.
- When: Verified on 2026-05-09 during beads task `groupscout-ui-561`.
- Why: The phase prompt pack still displayed unchecked supporting checklist items even though the canonical Phase 0-15 implementation and documentation were already present, and the audits found that production proxying needed to enforce the documented session boundary while Phase 15 route evidence could pass too much through navigation-only rendering.
- How: Claimed a beads issue, ran parallel read-only agent audits, added red-first tests for production request-handler session gating/security headers, route-specific Phase 15 evidence, and token drift, implemented the smallest server/renderer/token updates, reran focused tests, reran `npm test` and `npm run build`, confirmed `web/dist` had no tracked diff after build, updated Markdown status, and preserved the documented Phase 15 residual risk around real-browser screenshot/pixel checks.

### Phases 0-15 - GroupScout Web App Baseline And Browser Hardening

- What: Reconciled the checkout to the tracked UI baseline, completed the canonical Phase 0-15 prompt order, added backend compatibility coverage for `/api/leads/{id}/raw`, added the read-only lead detail client contract, and added deterministic Phase 15 browser UX hardening metadata for focus labels, accessible names, responsive variants, stable states, text containment, and same-origin API calls.
- Where: Added `test/baseline-reconciliation.test.js`, `test/browser-ux-hardening.test.js`, `web/src/renderer/browserUxHardening.js`, `docs/ui-tdd-phase-0-15-implementation.md`, and `docs/phase-15-browser-ux-hardening.md`; updated `web/src/api/leads.js`, `web/src/server/backendCompatibilitySmoke.js`, `test/api-boundary.test.js`, `test/lead-inbox-client.test.js`, `test/phase-13-renderer-runtime.test.js`, `README.md`, `UI_TDD_PHASE_PROMPTS.md`, `docs/testing.md`, `docs/developer-guide.md`, the retired supporting phase prompt checklist, and `docs/web-app-brainstorm.md`.
- When: Completed on 2026-05-09 after restoring the Phase 13 UI baseline and checking the canonical Phase 0-15 prompt pack.
- Why: The operator workbench needed a runnable baseline plus evidence that every requested UI TDD phase is represented by tests, implementation, docs, same-origin API boundaries, and server-side secret handling.
- How: Followed strict TDD with red runs for the absent baseline, missing raw-audit smoke route, missing `getLead` client, and missing Phase 15 hardening module; implemented the smallest contract changes; then reran focused tests, `npm run build`, `npm test`, Docker test-image build/run, production image build, and production-container smoke checks for `/healthz`, `/`, and `/assets/app.js`.

### Phase 13 - Product Renderer Runtime

- What: Implemented Phase 13-A through 13-F with a tested dependency-free vanilla DOM renderer/runtime contract, rendered route smoke harness, static product build, app-route fallback, product dev server Compose command, and backend compatibility smoke classification.
- Where: Added `test/phase-13-renderer-runtime.test.js`, `web/src/server/productRendererRuntime.js`, `web/src/renderer/domRenderer.js`, `web/src/renderer/buildStaticApp.js`, `web/src/server/productDevServer.js`, and `web/src/server/backendCompatibilitySmoke.js`; updated `package.json`, `compose.dev.yml`, `web/src/server/productionServer.js`, `web/dist/index.html`, `web/dist/assets/app.js`, `test/dockerization-contract.test.js`, `UI_TDD_PHASE_PROMPTS.md`, `README.md`, `docs/developer-guide.md`, `docs/testing.md`, `docs/troubleshooting.md`, `docs/phase-13-product-renderer-runtime.md`, and `docs/phase-13-product-renderer-runtime-prompts.md`.
- When: Completed on 2026-05-09 after Phase 12 D0-D5 Docker/runtime boundaries and the Phase 13 prompt pack.
- Why: Operators need the first real product renderer/runtime path while preserving the existing D4 same-origin static/proxy boundary and keeping backend/provider secrets out of browser-visible code, config, assets, Compose output, and CI artifacts.
- How: Followed strict TDD with a red `node test/phase-13-renderer-runtime.test.js` run, added the smallest contract modules and no-dependency renderer/build/dev-server code, regenerated static assets via `npm run build`, changed Compose only after product dev-server coverage existed, classified live backend route drift separately from proxy failures, then reran the focused Phase 13 suite, Dockerization contract suite, build, and full `npm test`.

### Docs - Phase 13 Renderer Runtime Prompt Pack

- What: Added a docs-only Phase 13 prompt pack with strict-TDD copy-paste prompts, parallel agent prompts, tickable phase tasks, backend Docker findings, UI working-tree findings, and explicit non-goals for future renderer/runtime work.
- Where: Added `docs/phase-13-product-renderer-runtime-prompts.md`; updated `docs/phase-13-product-renderer-runtime.md`, `UI_TDD_PHASE_PROMPTS.md`, `README.md`, and `docs/developer-guide.md`.
- When: Documented on 2026-05-09 after inspecting sibling backend Markdown/Docker files and current UI working-tree changes.
- Why: Future product renderer, browser test harness, static build, Compose, and live backend compatibility work needs phase-scoped prompts before any code, dependency, or Docker changes.
- How: Kept the pass documentation-only, preserved D1-D5 Docker boundaries, required red-first tests for every future implementation phase, and noted that current UI working-tree changes are limited to `.idea/*` line-ending churn.

### Docs - Backend Plus Frontend Docker E2E

- What: Clarified the current docs-only backend plus frontend Docker smoke path, including stable Compose project naming for the backend network and the distinction between D3 health harness and D4 static/proxy runtime.
- Where: Updated `docs/nice-to-knows.md`, `docs/how-to-run-backend.md`, `docs/docker-runtime-matrix.md`, `docs/developer-guide.md`, `docs/testing.md`, and `docs/troubleshooting.md`.
- When: Documented on 2026-05-09 while cross-checking backend planning docs and UI Docker files.
- Why: Developers need one predictable path for attaching `groupscout-ui-production` to `groupscout_groupscout_net` after starting the backend stack.
- How: Kept the pass documentation-only, added `-p groupscout` guidance, corrected the current `package.json` script summary, and preserved the no-secrets browser boundary.

### Docs - Docker Runtime Brainstorm

- What: Added a docs-only brainstorm for the next product renderer/runtime phase and a Docker runtime matrix that distinguishes the D1 test image, D3 development health harness, and D4 production static/proxy server.
- Where: Added `docs/phase-13-product-renderer-runtime.md` and `docs/docker-runtime-matrix.md`; updated `README.md`, `docs/developer-guide.md`, `docs/testing.md`, `docs/troubleshooting.md`, `docs/ui-dockerization-contract.md`, and `docs/phase-12-ui-dockerization.md`.
- When: Documented on 2026-05-09 after reviewing current Markdown, `Dockerfile`, `compose.dev.yml`, and Dockerization contract coverage.
- Why: Future renderer and Docker work needs a clearer planning surface without confusing the test image, Compose health harness, and production server behaviors.
- How: Kept the pass documentation-only, clarified stale D2/D4 wording, made the backend-Compose override requirement explicit, and captured future parallel-agent prompts for renderer fit, Docker impact, browser tests, and security boundaries.

### Phase 12 D5 - Docker Operations Docs And CI Hooks

- What: Added D5 Docker operations documentation and guardrail coverage for repeatable local tests, containerized tests, dev Compose startup/teardown, backend dependency expectations, required UI Docker env vars, troubleshooting, and future CI hook order.
- Where: Updated `test/dockerization-contract.test.js`, `docs/ui-dockerization-contract.md`, `docs/phase-12-ui-dockerization.md`, `UI_TDD_PHASE_PROMPTS.md`, `README.md`, `docs/developer-guide.md`, `docs/testing.md`, and `docs/troubleshooting.md`.
- When: Completed on 2026-05-09 as Phase 12 D5 after D4 production same-origin serving.
- Why: New developers and CI need a clear Docker operations path that distinguishes UI-only checks from backend-dependent Compose and proxy smoke checks without exposing automation credentials or backend secrets to browser-visible config.
- How: Followed strict TDD with a red `node test/dockerization-contract.test.js` run, added D5 docs/checklist assertions, documented command lifecycle, env var boundaries, CI order, and troubleshooting splits, then reran the focused Dockerization contract test, `npm test`, Docker test-image build/run, Compose config validation, production image build, and production-container smoke checks for `/healthz`, `/`, and `/assets/app.js`; `/api/system` was left backend-dependent because `localhost:8080` was not reachable.

### Phase 12 D4 - Same-Origin Proxy Or Static Serving

- What: Added D4 production same-origin serving with a lightweight Node server that serves `web/dist`, exposes `/healthz`, and forwards browser `/api/*` requests server-side to the backend target without exposing automation credentials.
- Where: Added `web/src/server/productionServer.js`, `web/dist/index.html`, and `web/dist/assets/app.js`; updated `Dockerfile`, `package.json`, `test/dockerization-contract.test.js`, `docs/ui-dockerization-contract.md`, `docs/phase-12-ui-dockerization.md`, `README.md`, `docs/developer-guide.md`, `docs/testing.md`, and `docs/troubleshooting.md`.
- When: Completed on 2026-05-09 as Phase 12 D4 after the D3 development Compose integration.
- Why: Browser users need one production origin for static UI assets and `/api/*` while `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, and `UI_SESSION_SECRET` stay server-side.
- How: Followed strict TDD with a red `node test/dockerization-contract.test.js` run, added D4 production server/static/proxy coverage, implemented `npm run start:ui` plus the `production` Docker target, documented smoke checks for `/healthz`, `/`, `/assets/app.js`, and `/api/system`, then reran the focused Dockerization contract test, `npm test`, Docker test-image build/run, and production image build.

### Phase 12 D3 - Development Compose Integration

- What: Added the D3 development Compose override and server-only health harness for a `groupscout-ui` service that joins the backend network, uses container port `3000`, healthchecks `/healthz`, and carries the internal `http://groupscout:8080` API target metadata without exposing browser or image secrets.
- Where: Added `compose.dev.yml` and `web/src/server/devComposeHealthServer.js`; updated `Dockerfile`, `test/dockerization-contract.test.js`, `docs/ui-dockerization-contract.md`, `docs/phase-12-ui-dockerization.md`, `UI_TDD_PHASE_PROMPTS.md`, `README.md`, `docs/developer-guide.md`, `docs/testing.md`, and `docs/troubleshooting.md`.
- When: Completed on 2026-05-09 as Phase 12 D3 after the D2 browser runtime contract.
- Why: Developers need a validated UI Compose integration point beside the backend stack before production same-origin proxying, static asset serving, or a rendered browser runtime is introduced.
- How: Followed strict TDD with a red `node test/dockerization-contract.test.js` run, added the Compose override plus D3 health metadata, documented commands and scope boundaries, then reran the focused Dockerization contract test, `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`, local `npm test`, `docker build --target test -t groupscout-ui-test .`, and `docker run --rm groupscout-ui-test`.

### Phase 12 D2 - Browser Runtime Contract

- What: Added the D2 browser runtime contract for a future lightweight Node UI server, including reserved `npm run start:ui`, container port `3000`, `/healthz`, static asset boundary metadata, same-origin `/api/*` routing expectations, and forbidden browser public config checks.
- Where: Added `web/src/server/browserRuntimeContract.js`; updated `test/dockerization-contract.test.js`, `docs/ui-dockerization-contract.md`, `docs/phase-12-ui-dockerization.md`, `UI_TDD_PHASE_PROMPTS.md`, `README.md`, `docs/developer-guide.md`, `docs/testing.md`, and `docs/troubleshooting.md`.
- When: Completed on 2026-05-09 as Phase 12 D2 after the D1 UI test container.
- Why: Future Compose and runtime work needs a tested browser runtime contract before adding a dev server, renderer, framework, proxy, or runnable UI server.
- How: Followed strict TDD with a red `node --test test/dockerization-contract.test.js` run, added server-side contract metadata and public-config guard helpers, documented the runtime shape and evidence, then reran the focused Dockerization contract test and full `npm test`.

### Phase 12 D1 - UI Test Container

- What: Added the first Docker implementation target for the UI repo: a deterministic Node test image that runs the current `npm test` suite, plus guardrail coverage for Dockerfile, `.dockerignore`, no-runtime scope, and credential boundaries.
- Where: Added `Dockerfile` and `.dockerignore`; updated `test/dockerization-contract.test.js`, `docs/ui-dockerization-contract.md`, `docs/phase-12-ui-dockerization.md`, `UI_TDD_PHASE_PROMPTS.md`, `README.md`, `docs/developer-guide.md`, and `docs/testing.md`.
- When: Completed on 2026-05-09 as Phase 12 D1 after the D0 Dockerization contract.
- Why: The UI model-level workspace needs a clean containerized test path before any browser runtime, dev server, reverse proxy, or Compose integration is introduced.
- How: Followed strict TDD with a red `node test/dockerization-contract.test.js` run, added a single `test` Docker target and context exclusions, documented `docker build --target test -t groupscout-ui-test .` plus `docker run --rm groupscout-ui-test`, then reran the focused test, Docker build/container test, and local `npm test`.

### Phase 12 D0 - Dockerization Contract And Decision Record

- What: Added the documentation-only D0 Dockerization contract and a focused guardrail test that makes the contract testable before Docker files exist.
- Where: Added `docs/ui-dockerization-contract.md` and `test/dockerization-contract.test.js`; updated `docs/phase-12-ui-dockerization.md`, `UI_TDD_PHASE_PROMPTS.md`, `README.md`, `docs/developer-guide.md`, and `docs/testing.md`.
- When: Completed on 2026-05-09 as the first implementation task in the Phase 12 UI dockerization sequence.
- Why: Future Docker work needs an explicit path, backend service contract, and browser security boundary before adding images, Compose, proxies, or runtimes.
- How: Followed D0 strict TDD with a red `node --test test/dockerization-contract.test.js` run, documented the test-image-first decision, backend internal URLs, same-origin `/api/*` rule, and no-`API_TOKEN` browser constraint, then reran the focused test and full `npm test`.

### Phase 12 - UI Dockerization Planning

- Added a planning-only strict-TDD prompt pack for future UI dockerization in `docs/phase-12-ui-dockerization.md`.
- Captured backend Docker/Compose constraints from the sibling Go repo and current UI gaps: no Dockerfile, Compose file, renderer, dev server, build tool, lockfile, or browser runtime yet.
- Updated `UI_TDD_PHASE_PROMPTS.md`, `README.md`, `docs/developer-guide.md`, and `docs/testing.md` so future implementation starts with a Docker contract and test image before adding runtime or Compose behavior.

### Smell Phase H1 - Split The Growing API Client

- What: Split the browser API client into focused adapter modules while preserving the stable `createApiClient(...)` facade, public constants, endpoint paths, payload normalization, response adaptation, and centralized same-origin transport guard.
- Where: Updated `web/src/api/client.js`, added `web/src/api/transport.js`, `web/src/api/shared.js`, `web/src/api/leads.js`, `web/src/api/rawAudit.js`, `web/src/api/outreach.js`, `web/src/api/pipeline.js`, `web/src/api/stats.js`, `web/src/api/alerts.js`, `web/src/api/system.js`, and tightened `test/api-boundary.test.js`.
- When: Completed on 2026-05-09 after the H0 API-client characterization baseline.
- Why: Reduced review risk in the growing API client module without changing the browser-facing import path or weakening the security-sensitive `/api/*` same-origin boundary.
- How: Added split-module ownership coverage, moved feature-specific route builders/payload builders/adapters behind focused modules, kept transport validation in `transport.js`, documented the module map in `docs/smell-h1-api-client-split.md`, marked H1 complete in `docs/code-smell-transformation-prompts.md`, and reran the focused API-client suite plus `npm test`.

### Smell Phase H0 - API Client Baseline Characterization

- What: Added H0 characterization coverage for the browser API client public exports, `createApiClient(...)` method surface, same-origin transport defaults, invalid-route pre-fetch rejection, encoded lead-scoped routes, payload defaults, adapter defaults, and read-only policy metadata.
- Where: Updated `test/api-boundary.test.js`, `test/lead-inbox-client.test.js`, `test/lead-status-mutation-client.test.js`, `test/raw-audit-client.test.js`, `test/outreach-client.test.js`, `test/pipeline-client.test.js`, `test/stats-client.test.js`, `test/alert-client.test.js`, and `test/system-client.test.js`.
- When: Completed on 2026-05-09 before starting the H1 API-client split.
- Why: Created a stable baseline so the H1 refactor fails on real contract drift instead of import-path or file-layout movement.
- How: Tightened model-level `node:test` assertions, documented the affected H1 behavior map in `docs/smell-h0-api-client-characterization.md`, marked H0 complete in `docs/code-smell-transformation-prompts.md`, and refreshed README/testing/developer docs with the focused baseline command.

### Phase 11 - Today Command Center And System Health Summary

- Added failing-first Phase 11 tests for the Today command center, priority lead and aging-work summaries, active alerts, failed jobs, system health, read-only action policy, responsive metadata, `DESIGN.md` token usage, `/` route mounting, and `GET /api/system`.
- Added `web/src/app/todayCommandCenter.js` with mocked command-center data, summary cards, dense work rows, cross-workspace navigation intents, loading/empty/error states, responsive layouts, and read-only policy metadata.
- Added `createApiClient().getSystem()` for same-origin read-only `GET /api/system` access.
- Mounted the Today command center from `createRouteShell("/")` while keeping Settings as a placeholder.
- Added Phase 11 implementation notes in `docs/phase-11-today-command-center-system-health.md`, `README.md`, `docs/testing.md`, `docs/developer-guide.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Phase 10 - Later Alertd Read-Only Console

- Added failing-first Phase 10 tests for read-only alert state rendering, SPS summaries, evidence display, room inventory, action history, disabled mutation actions, `/alerts` route mounting, responsive metadata, `DESIGN.md` token usage, and `GET /api/alerts`.
- Added `web/src/app/alertdConsole.js` with mocked Alertd data, Slack-first policy metadata, dense alert rows, evidence rows, room inventory details, action-history rows, loading/empty/error states, responsive layouts, and disabled acknowledge/resolve/suppress actions.
- Added `createApiClient().listAlerts(...)` for read-only same-origin `GET /api/alerts` access.
- Mounted the Alertd console from `createRouteShell("/alerts")`.
- Added Phase 10 implementation notes in `docs/phase-10-later-alertd-read-only-console.md`, `README.md`, `docs/testing.md`, `docs/developer-guide.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Phase 9 - Session/Auth Wrapper And Same-Origin Deployment

- Added failing-first Phase 9 tests for session-required `/api/*` access, recursive browser-source `API_TOKEN` exclusion, base-path mounting, disabled UI behavior, deployment readiness, and development-only CORS configuration.
- Added `web/src/server/uiDeployment.js` with `UI_ENABLED`, `UI_BASE_PATH`, `UI_SESSION_SECRET`, and `CORS_ALLOWED_ORIGINS` config parsing plus session-cookie API authorization.
- Added `createMountedRouteShell(...)` so deployed UI routes under `UI_BASE_PATH` map back to workspace routes and expose base-path-aware navigation hrefs.
- Kept browser API requests same-origin with session credentials and no automation-token headers.
- Added Phase 9 implementation notes in `docs/phase-9-session-auth-wrapper-same-origin-deployment.md`, `README.md`, `docs/testing.md`, `docs/developer-guide.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Phase 8 - Basic Analytics And Demand Signals

- Added failing-first Phase 8 tests for `GET /api/stats`, status/source/score-band/owner/week summaries, source-yield hit-rate definitions, lead aging, verification quality, upcoming demand, denominator/date-range labels, `/analytics` route mounting, responsive metadata, and `DESIGN.md` token usage.
- Added `web/src/app/analyticsDashboard.js` with mocked analytics stats, explainable source-yield calculations, lead aging, verification quality, demand rows, empty/loading/error states, and responsive layout metadata.
- Added `createApiClient().getStats(...)` for same-origin `GET /api/stats` with date range, segment, and property filters.
- Mounted the Analytics screen from `createRouteShell("/analytics")`.
- Added Phase 8 implementation notes in `docs/phase-8-basic-analytics-demand-signals.md`, `README.md`, `docs/testing.md`, `docs/developer-guide.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Phase 7 - Pipeline Monitor And Run Controls

- Added failing-first Phase 7 tests for pipeline run history, async run creation, compact collector/LLM/delivery health fields, recent failures, partial-data states, `/pipeline` route mounting, same-origin API boundaries, and `DESIGN.md` token usage.
- Added `web/src/app/pipelineMonitor.js` with mocked pipeline run data, compact health summaries, recent failure summaries, optional log/Grafana links, responsive layouts, and browser-async run-control metadata.
- Added `createApiClient().listPipelineRuns(...)` and `createApiClient().startPipelineRun(...)` for same-origin `GET/POST /api/pipeline/runs`.
- Mounted the Pipeline Monitor from `createRouteShell("/pipeline")`.
- Added Phase 7 implementation notes in `docs/phase-7-pipeline-monitor-run-controls.md`, `README.md`, `docs/testing.md`, `docs/developer-guide.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Phase 6 - Outreach Workspace And Activity Log

- Added failing-first Phase 6 tests for editable outreach drafts, contact validation, manual copied/sent/logged states, same-origin outreach API reads/writes, outcome capture, Lead Detail activity rows, `/outreach` route mounting, responsive metadata, and `DESIGN.md` token usage.
- Added `web/src/app/outreachWorkspace.js` with mocked outreach lead data, editable draft/contact field models, manual display states, outcome options, timeline rendering, responsive layouts, and no-auto-send/no-CRM-sync policy metadata.
- Added `createApiClient().listLeadOutreach(...)` and `createApiClient().logLeadOutreach(...)` for same-origin `GET/POST /api/leads/{id}/outreach`.
- Embedded a compact manual outreach workspace in Lead Detail and extended the activity log with outreach attempt and outcome entries.
- Mounted the Outreach Workspace from `createRouteShell("/outreach")`.
- Added Phase 6 implementation notes in `docs/phase-6-outreach-workspace-activity-log.md`, `README.md`, `docs/testing.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Phase 5 - Verification Queue And Raw Audit Review

- Added failing-first Phase 5 tests for verification trigger classification, queue filters/actions, `/verification` route integration, raw audit API alias use, responsive metadata, and `DESIGN.md` token usage.
- Added `web/src/app/verificationQueue.js` with trigger classification, mocked queue data, dense row models, mobile card metadata, row actions, raw audit link intent, and an explicit blocked redaction-policy TODO.
- Added `createApiClient().getLeadRawAudit(...)` for same-origin `GET /api/leads/{id}/raw` access and updated Lead Detail raw audit links to use the UI-safe alias.
- Mounted the Verification Queue from `createRouteShell("/verification")`.
- Added Phase 5 implementation notes in `docs/phase-5-verification-queue-raw-audit-review.md`, `README.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Housekeeping Docs

- Added UI developer, testing, troubleshooting, nice-to-know, backend spin-up, and code-smell housekeeping docs under `docs/`.
- Documented backend startup paths from the sibling Go repo without changing backend code.
- Captured current UI test-coverage limits and known doc/config drift to guide future cleanup.
- Refreshed housekeeping docs with recursive credential-guard behavior, current phase links, focused test commands, optional design linting, and the latest code-smell watchlist.
- Added strict-TDD code-smell transformation prompts with tickable housekeeping phases and tasks for future refactor work.

### Phase 4 - Lead Status Actions And State Model

- Added failing-first Phase 4 state model tests for every recommended v1 status, allowed action, and disallowed transition.
- Added `web/src/app/leadStatus.js` with status constants, transition helpers, action metadata, validation, PATCH mutation intent building, and auditable field correction helpers.
- Added Lead Detail action controls that show only valid actions for the current lead status and reject invalid transitions before calling the API mutation boundary.
- Added `createApiClient().patchLead(...)` for same-origin `PATCH /api/leads/{id}` payloads covering status, owner, notes, snooze date, correction reason, and safe field corrections.
- Updated Phase 4 docs in `docs/phase-4-lead-status-actions-state-model.md`, `README.md`, and `UI_TDD_PHASE_PROMPTS.md`.

### Phase 3 - Lead Detail Evidence Workspace

- Added failing-first Phase 3 Lead Detail screen tests for required evidence sections, source evidence, raw audit link intent, AI enrichment rationale/uncertainty, activity timeline entries, reviewer correction distinction, state handling, responsive metadata, and `DESIGN.md` token usage.
- Added `createLeadDetailScreen(...)` and mocked lead detail evidence under `web/src/app/leadDetail.js`.
- Mounted read-only lead detail workspaces from `/leads/{id}` while keeping the Leads navigation section active.
- Expanded design token exports with documented `DESIGN.md` component names needed by the Lead Detail workspace: `card-base` and `code-inline`.
- Added Phase 3 implementation notes in `docs/phase-3-lead-detail-evidence-workspace.md`.

### Phase 2 - Lead Inbox UI

- Added failing-first Phase 2 Lead Inbox screen tests for dense table rendering, mocked lead rows, filters, loading/empty/error states, row detail navigation, responsive layout metadata, accessibility metadata, and `DESIGN.md` token usage.
- Added `createLeadInboxScreen(...)`, Lead Inbox columns, and mocked lead data under `web/src/app/leadInbox.js`.
- Mounted the Lead Inbox screen from the `/leads` route shell while keeping non-lead routes as placeholders.
- Expanded design token exports with documented `DESIGN.md` component names needed by the Lead Inbox: `button-secondary`, `search-pill`, `badge-tag`, `badge-type`, `feature-comparison-table`, and `property-row`.
- Added Phase 2 implementation notes in `docs/phase-2-lead-inbox-ui.md`.

### Phase 1 - Lead Inbox API Contract And Client

- Added Phase 1 lead inbox API contract tests for `GET /api/leads` query serialization, pagination, default priority ordering, response field adaptation, and missing-field validation.
- Added `createApiClient().listLeads(...)` as the typed browser boundary for the future lead inbox UI.
- Added default lead inbox priority sort metadata for urgent, unowned, high-score leads.
- Added Phase 1 contract documentation in `docs/phase-1-lead-inbox-contract.md`.

### Phase 0 - Product Contract And Test Harness

- Added a no-dependency Node test harness with `npm test`.
- Added Phase 0 guardrail tests for `DESIGN.md` token mapping, route shell IA, `/api/*` client isolation, and browser credential safety.
- Added minimal design token exports under `web/src/design/tokens.js`.
- Added a placeholder operator workspace shell under `web/src/app/shell.js`.
- Added a same-origin `/api/*` browser client boundary under `web/src/api/client.js`.
- Added Phase 0 implementation notes in `docs/phase-0-product-contract.md`.
