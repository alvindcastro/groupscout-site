# Developer Guide

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/frontend/docs/developer-guide.md`. Run UI implementation, build, test, and Docker commands from the UI source repo at `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

The UI source repo is currently a plain JavaScript UI workspace for GroupScout operator screens. It has model-level screen factories, a dependency-free vanilla DOM renderer, a static build script, and Node's built-in test runner.

## Current Shape

- `web/src/api/client.js` is the stable browser API facade. Focused adapters live under `web/src/api/*`, and `web/src/api/transport.js` owns the shared same-origin `/api/*` request guard.
- `web/src/app/shell.js` owns route-shell selection.
- `web/src/app/adminLogin.js` owns the setup-token login screen model.
- `web/src/server/uiDeployment.js` owns model-level UI deployment settings, base-path mounting, session-cookie API authorization, and development-only CORS metadata.
- `web/src/server/browserRuntimeContract.js` owns the D2 browser runtime contract metadata for the future lightweight Node server, reserved port, health path, static asset boundary, `/api/*` routing expectation, and forbidden browser public config keys.
- `web/src/server/productRendererRuntime.js` owns the Phase 13 renderer/runtime contract.
- `web/src/server/productDevServer.js` owns the Phase 13 product dev server for the `groupscout-ui` Compose service.
- `web/src/server/productionServer.js` owns the D4 production same-origin server for `web/dist` static assets, `/healthz`, whitelist-only public config, and server-side `/api/*` proxying.
- `web/src/renderer/domRenderer.js` owns the first dependency-free rendered route mapping.
- `web/src/renderer/staticAppEntry.js` owns the browser static entrypoint, app-route-only click interception, and dynamic loading of copied renderer modules.
- `web/src/renderer/browserUxHardening.js` owns the Phase 15 deterministic browser UX hardening report until a real browser harness is added.
- `web/src/renderer/buildStaticApp.js` owns the static product build into `web/dist`.
- `web/src/app/todayCommandCenter.js` owns the mocked Today command center, operational priority summaries, system health metadata, and read-only routing policy.
- `web/src/app/leadInbox.js` owns the mocked Lead Inbox screen model.
- `web/src/app/leadDetail.js` owns the mocked Lead Detail Evidence Workspace.
- `web/src/app/leadStatus.js` owns status/action transition rules and mutation intent construction.
- `web/src/app/verificationQueue.js` owns the mocked Verification Queue screen model and raw audit review metadata.
- `web/src/app/outreachWorkspace.js` owns the mocked Outreach Workspace screen model and manual outreach logging metadata.
- `web/src/app/pipelineMonitor.js` owns the mocked Pipeline Monitor screen model, health summaries, and async run-control metadata.
- `web/src/app/analyticsDashboard.js` owns the mocked Analytics screen model, source-yield definitions, and demand-signal metadata.
- `web/src/app/alertdConsole.js` owns the mocked Alertd read-only console model, SPS summaries, evidence, room inventory, action-history metadata, and disabled alert action policy.
- `web/src/design/tokens.js` exports the subset of `DESIGN.md` tokens needed by tests.
- `test/*.test.js` contains contract and screen-model tests using `node:test`.

There is no bundler, framework runtime, lockfile, or package-install step yet.

## Coordinator Workflow

- Start feature and bug-fix prompts in `/mnt/c/Users/alvin/groupscout-site`.
- Read and update long-lived markdown under `frontend/` and `backend/` in the coordinator repo.
- Make UI code changes in `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.
- Make backend code changes in `/mnt/c/Users/alvin/GolandProjects/groupscout`.
- Create or reuse a source-repo task branch before editing implementation files when possible.
- Keep generated or tool-owned docs, such as Beads metadata, in their owning repo.

## Runtime

Use Node `18+` or newer. Tests rely on modern built-in web APIs such as `Response.json`.

## Daily Commands

Local UI tests:

```sh
npm test
```

Containerized test-image run:

```sh
docker build --target test -t groupscout-ui-test .
docker run --rm groupscout-ui-test
```

Use [Docker Runtime Matrix](./docker-runtime-matrix.md) to choose between the D1 test image, Phase 13 development Compose product server, and D4 production static/proxy server. It owns current Compose commands, backend-network smoke checks, expected `/api/*` statuses, port rules, and secret boundaries.

Production same-origin server:

```sh
npm run start:ui
```

Production cache and navigation rules:

- `/assets/*` files are treated as immutable versioned assets.
- `/src/*` copied renderer modules are served with `cache-control: no-store` because they are unbundled and may be imported transitively without a query string.
- Bump the static asset query in `buildStaticApp.js` and `staticAppEntry.js` when Docker smoke needs browsers to stop using an older immutable app bundle; the admin-login Docker refresh uses `admin-login-2`.
- Static app click interception is limited to first-party app routes. `/api/*` raw audit links, `/assets/*`, `/src/*`, file-extension URLs, download links, target links, external links, and modified clicks keep normal browser behavior.

Optional design-doc lint. This uses `npx` and may fetch the package if it is not already cached:

```sh
npx @google/design.md lint DESIGN.md
```

Useful focused runs:

```sh
node --test test/api-boundary.test.js
node --test test/app-shell.test.js
node --test test/design-tokens.test.js
node --test test/lead-inbox-client.test.js
node --test test/lead-inbox-screen.test.js
node --test test/lead-detail-screen.test.js
node --test test/lead-status-state-model.test.js
node --test test/lead-status-mutation-client.test.js
node --test test/verification-queue.test.js
node --test test/raw-audit-client.test.js
node --test test/outreach-client.test.js
node --test test/outreach-workspace.test.js
node --test test/pipeline-client.test.js
node --test test/pipeline-monitor.test.js
node --test test/stats-client.test.js
node --test test/analytics-dashboard.test.js
node --test test/analytics-screen.test.js
node --test test/alert-client.test.js
node --test test/alert-console.test.js
node --test test/system-client.test.js
node --test test/today-command-center.test.js
node --test test/session-deployment.test.js
node --test test/dockerization-contract.test.js
node --test test/baseline-reconciliation.test.js
node --test test/browser-ux-hardening.test.js
```

Admin login floating-window verification after CSS or renderer changes:

```sh
node --test test/phase-13-renderer-runtime.test.js
npm run build
npm test
```

Manual Docker/browser smoke should open `http://localhost:3001/admin/login` and confirm the login form appears as a compact floating window, not as a full-width page panel. The automated renderer test asserts the `admin-login-window` class contract; it is not a visual regression test.

API-client smell-phase baseline:

```sh
node --test test/api-boundary.test.js test/lead-inbox-client.test.js test/lead-status-mutation-client.test.js test/raw-audit-client.test.js test/outreach-client.test.js test/pipeline-client.test.js test/stats-client.test.js test/alert-client.test.js test/system-client.test.js
```

## Development Rules

- Keep browser requests behind same-origin `/api/*` paths.
- Keep same-origin `/api/*` links outside SPA route interception so raw audit and other API links reach the production proxy normally.
- Keep browser auth session-based. Do not repurpose automation credentials for operator browser sessions.
- Keep protected app routes behind `/api/auth/status` checks and redirect unauthenticated users to `/admin/login` when auth is required.
- Keep setup-token handling server-owned; see [Admin Token Flow](./admin-token-flow.md) for token source, rotation, logout, stale asset recovery, and direct curl smoke commands.
- Keep the admin login UI as a compact `admin-login-window` form inside the `admin-login` centering stage; do not move setup-token entry into a generic full-width screen panel.
- Keep logout wired through `createApiClient().logout()` and `/api/auth/logout`; do not clear cookies from browser JavaScript directly.
- Keep `API_TOKEN` out of browser runtime/config modules.
- Keep `UI_API_PROXY_TARGET` server-only; browser code and public config may only expose relative `/api/*`.
- Keep production public config whitelist-only and free of provider keys, Slack tokens, Resend/SendGrid keys, database URLs, and `UI_SESSION_SECRET`.
- Configure `UI_SESSION_SECRET` for real operator deployments so browser `/api/*` requests require a valid `groupscout_session`; leave it unset only for backend Docker smoke checks that need proxy reachability without a browser login flow.
- Use `UI_ENABLED` to disable the UI, `UI_BASE_PATH` for subpath mounting, and `UI_SESSION_SECRET` for session readiness.
- Keep `CORS_ALLOWED_ORIGINS` development-only; same-origin deployment is the default production posture.
- Add API access through `createApiClient(...)`; do not fetch backend URLs directly from app modules.
- Use `createApiClient().getLead(...)` for browser-facing `GET /api/leads/{id}` detail reads; keep source evidence, AI enrichment, reviewer corrections, and activity distinct.
- Use `GET /api/leads/{id}/raw` for browser-facing raw audit access; do not link older raw audit endpoints from UI screens.
- Use `GET/POST /api/pipeline/runs` for pipeline history and manual run creation; do not call worker, scheduler, or one-shot automation endpoints directly from app modules.
- Use `GET /api/stats` for browser-facing analytics; keep denominators, date ranges, and outcome definitions visible when adding metrics.
- Use `GET /api/alerts` for the read-only Alertd console; keep Slack as the interrupt channel and keep alert mutations disabled unless a future contract explicitly adds them.
- Use `GET /api/system` for Today/system-health summaries; keep Today as a read-only routing surface and leave mutations in the owning workspaces.
- Keep status transitions in `leadStatus.js`; UI surfaces should consume action metadata instead of duplicating the transition table.
- Keep mocked screen data aligned across inbox and detail models when a mocked lead appears in both places.
- Treat `DESIGN.md` as the source design contract; `web/src/design/tokens.js` is a partial implementation used by tests.
- Update phase docs and README when behavior or scope changes.

## Docker Operations

Docker operation mode selection, required env vars, backend dependency expectations, and D3/D4 smoke commands live in [Docker Runtime Matrix](./docker-runtime-matrix.md).

The production UI Compose profile and repeatable backend plus UI Docker E2E gate are implemented; run `make smoke-ui-docker-e2e` from the backend repo.

CI hook order:

1. `npm test`
2. `docker build --target test -t groupscout-ui-test .`
3. `docker run --rm groupscout-ui-test`
4. `docker build --target production -t groupscout-ui-production .`
5. Optional smoke checks for `/healthz`, `/`, and `/assets/app.js`; smoke `/api/system` only when a backend or CI stub is reachable, and expect the backend's current status through the proxy.

Do not run UI Docker containers with backend `.env` or `--env-file`. Do not pass `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` into browser-visible config, static assets, Compose output, or CI artifacts. Browser-visible config may only expose relative `/api/*`.

## Backend Integration Boundary

The backend repo is separate:

```sh
/mnt/c/Users/alvin/GolandProjects/groupscout
```

For backend startup and API checks, see [how-to-run-backend.md](./how-to-run-backend.md).

For the exact operator login, setup-token rotation, API smoke, and stale-asset recovery flow, see [admin-token-flow.md](./admin-token-flow.md).

Backend markdown references are centralized under `/mnt/c/Users/alvin/groupscout-site/backend`; do not expect backend docs to exist in the backend source checkout after the doc move.

Current UI client contracts:

- `GET /api/leads`
- `GET /api/leads/{id}`
- `PATCH /api/leads/{id}`
- `GET /api/leads/{id}/raw`
- `GET /api/leads/{id}/outreach`
- `POST /api/leads/{id}/outreach`
- `GET /api/pipeline/runs`
- `POST /api/pipeline/runs`
- `GET /api/stats`
- `GET /api/alerts`
- `GET /api/system`

## API Module Map

- `web/src/api/client.js`: public facade for `createApiClient(...)` and exported constants.
- `web/src/api/auth.js`: admin auth status, setup-token login, current-admin lookup, and logout methods.
- `web/src/api/transport.js`: centralized same-origin `/api/*` request validation, JSON defaults, session credentials, non-2xx errors, and non-JSON success behavior.
- `web/src/api/shared.js`: small helpers shared by adapters.
- `web/src/api/leads.js`: lead inbox reads, lead detail reads, and lead PATCH writes.
- `web/src/api/rawAudit.js`: raw audit reads.
- `web/src/api/outreach.js`: outreach history reads and manual outreach attempt logging.
- `web/src/api/pipeline.js`: pipeline run history and manual run creation.
- `web/src/api/stats.js`: analytics stats reads and hit-rate metadata adaptation.
- `web/src/api/alerts.js`: read-only alert list reads.
- `web/src/api/system.js`: read-only system summary reads.

## Centralized Doc Tests

Some tests still assert against long-lived markdown contracts after docs moved to `/mnt/c/Users/alvin/groupscout-site/frontend`. They first read local docs when present, then fall back to `GROUPSCOUT_UI_DOCS_ROOT` or `/mnt/c/Users/alvin/groupscout-site/frontend`.

## Adding A New UI Phase

1. Write a focused test that describes the contract or screen-model behavior first.
2. Keep the first implementation narrow and model-level unless the phase explicitly introduces rendering.
3. Update the relevant `docs/phase-*` file or create a new phase doc.
4. Update `README.md` current scope.
5. Update `CHANGELOG.md` under `Unreleased`.
6. Run `npm test`.

## Transforming Code Smells

Use [code-smell-transformation-prompts.md](./code-smell-transformation-prompts.md) for future housekeeping refactors. Treat each smell phase like a product phase: characterize behavior first, prove the red state, make the smallest change, rerun focused tests plus `npm test`, then update docs.

The H0 baseline is [smell-h0-api-client-characterization.md](./smell-h0-api-client-characterization.md), and H1 completion is documented in [smell-h1-api-client-split.md](./smell-h1-api-client-split.md). H2, isolating mock fixtures from runtime factories, is the next active smell phase.

## Dockerization Planning

Use [phase-12-ui-dockerization.md](./phase-12-ui-dockerization.md) for the phased prompt pack, [UI Dockerization Contract](./ui-dockerization-contract.md) for the D0-D5 decision record, and [Docker Runtime Matrix](./docker-runtime-matrix.md) when deciding which container mode to run. The repo has a minimal `Dockerfile` test target and `.dockerignore`; the test image runs `npm test` without a package install step or backend service.

Runtime model: `lightweight-node-server`. D4 implements `npm run start:ui`, container port `3000`, health path `/healthz`, server-owned assets under `web/dist`, app-route fallback to `index.html`, and server-side `/api/*` routing to `http://groupscout:8080` by default. Phase 13 adds a dependency-free vanilla DOM renderer and static build while still avoiding framework and lockfile changes.

The Phase 13 implementation record is [Phase 13 Product Renderer Runtime](./phase-13-product-renderer-runtime.md), with detailed prompts and task status in [Phase 13 Product Renderer Runtime Prompt Pack](./phase-13-product-renderer-runtime-prompts.md). It treats renderer choice, development-server behavior, browser-level tests, static asset safety, and backend compatibility smoke coverage as one decision set so the D0-D5 Docker contract remains stable.

Static product build:

```sh
npm run build
```

Development Compose and D4 production smoke details live in [Docker Runtime Matrix](./docker-runtime-matrix.md). The matrix owns the current command blocks, the `-p groupscout` network convention, the `${GROUPSCOUT_UI_HOST_PORT:-3001}` default, backend dependency expectations, and the D4 `/api/leads`, `/api/system`, and `/api/alerts` smoke checks.

## Current Limitations

- Tests now include dependency-free rendered HTML smoke coverage, but not a real browser engine.
- Accessibility checks cover rendered landmarks, labels, and focusable metadata; they do not perform full assistive-technology audits.
- Responsive behavior is still represented as layout metadata, not measured CSS layout.
- Design token tests check selected exports; they do not resolve every nested token reference from `DESIGN.md`.
- The browser credential guard recursively scans browser-facing `web/src/**/*.js` files while excluding `web/src/server`; keep server-only code in that excluded subtree and keep browser modules free of automation credentials.
- Framework rendering remains a future strict-TDD phase; the current renderer is a minimal vanilla DOM bridge over the existing screen models.
