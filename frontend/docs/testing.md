# Testing

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/frontend/docs/testing.md`. Run UI commands from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui` and backend commands from `/mnt/c/Users/alvin/GolandProjects/groupscout`.

## UI Repo

Run the full UI test suite:

```sh
npm test
```

The command maps to:

```sh
node --test
```

There is still no package install requirement.

## Current Docker Test Prep

As of 2026-05-09, the local Docker stack is ready for UI smoke and backend integration testing:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
docker compose up -d postgres

cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui
docker compose -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  ps
```

Current expected containers:

- `groupscout_postgres`: healthy on `localhost:5432`.
- `groupscout_app`: backend API on `localhost:8080`.
- `groupscout-groupscout-ui-1`: UI product dev server on `localhost:3001`, healthy.
- `groupscout_n8n`: available on `localhost:5678` if workflow testing is needed.
- `groupscout_ollama`: container-healthy, but backend `/health` can still report `"ollama":"unavailable"` until API-to-Ollama connectivity/model readiness is fixed.

Smoke before testing:

```sh
curl -i --max-time 5 http://localhost:8080/health
curl -i --max-time 5 http://localhost:3001/healthz
curl -i --max-time 5 http://localhost:3001/
```

Expected current prep result: backend `/health` returns `200` with `"database":"ok"`, UI `/healthz` returns `200`, and the UI shell responds. The Ollama health caveat does not block ordinary UI tests, static/proxy checks, or Postgres-backed API tests; it does block confidence in LLM/enrichment behavior.

Admin login Docker smoke refresh on 2026-05-09:

```sh
cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui
npm run build
docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up -d --build groupscout-ui
curl -i --max-time 5 http://localhost:3001/api/auth/status
curl -i --max-time 5 http://localhost:3001/
```

Expected: `/api/auth/status` returns `auth_required:true` and `authenticated:false`, and `/` references `/assets/app.js?v=admin-login-2`. If the browser still shows the old app without login, hard-refresh or open `/admin/login` directly so the immutable old `pipeline-output-4` asset is not reused from browser cache.

The UI Docker test image no longer copies `DESIGN.md` or `docs/` from the UI repo because long-lived markdown lives in `/mnt/c/Users/alvin/groupscout-site/frontend`. In an isolated container without that mount, centralized-doc-only assertions are skipped while runtime/code assertions still run. To force doc assertions inside the container, mount the coordinator docs and set `GROUPSCOUT_UI_DOCS_ROOT`.

Admin token flow smoke:

```sh
TOKEN="$(docker exec groupscout_app sh -lc 'cat data/admin-setup-token')"
curl -i -c /tmp/groupscout-admin.cookies \
  -H "Content-Type: application/json" \
  -d "{\"token\":\"${TOKEN}\"}" \
  http://localhost:3001/api/auth/login
curl -i -b /tmp/groupscout-admin.cookies http://localhost:3001/api/auth/me
curl -i -b /tmp/groupscout-admin.cookies -c /tmp/groupscout-admin.cookies \
  -X POST http://localhost:3001/api/auth/logout
```

The token file is `/app/data/admin-setup-token` inside the `groupscout_app` container; the host command above reads it through `docker exec`. File-backed setup tokens rotate after successful login. Read the file again if another login is needed. Env-backed `ADMIN_SETUP_TOKEN` values do not rotate automatically.

Current verification refresh on 2026-05-09: `npm test` passed all 26 test files after the renderer runtime review fixes for raw-audit-safe SPA navigation, `/src/*` cache policy, and mobile Verification Queue rendering.

Admin login completion run on 2026-05-09: `node --test test/app-shell.test.js test/api-boundary.test.js test/phase-13-renderer-runtime.test.js test/session-deployment.test.js`, `npm run build`, and `npm test` passed after adding protected-route auth redirects, setup-token login verification, logout, production readiness enforcement, and centralized-doc test fallbacks.

Admin login floating-window refresh on 2026-05-09: `npm test`, `npm run build`, and a second `npm test` passed after changing `/admin/login` so the form renders with `class="admin-login-window"` inside a transparent centering stage instead of a full-width panel. Use `node --test test/phase-13-renderer-runtime.test.js` for the focused renderer contract and `npm test` before pushing. The current tests assert the rendered class contract; they do not prove pixel layout, so manual browser smoke should confirm the login appears as a compact floating window.

Renderer runtime review fix on 2026-05-09: `node --test test/phase-13-renderer-runtime.test.js test/dockerization-contract.test.js test/verification-queue.test.js` covered app-route-only click interception, normal `/api/*` raw audit link behavior, copied `/src/*` module `no-store` caching, and mobile Verification Queue cards.

Housekeeping run on 2026-05-08: `npm test` passed all 22 test files.

Smell H0 baseline run on 2026-05-09: focused API-client characterization passed 9 test files, and `npm test` passed all 22 test files.

Smell H1 split run on 2026-05-09: focused API-client split coverage passed 9 test files, and `npm test` passed all 22 test files.

Phase 12 D0 run on 2026-05-09: `node --test test/dockerization-contract.test.js` covered the documentation-only Dockerization contract before any Docker files were added, and `npm test` passed all 23 test files.

Phase 12 D1 run on 2026-05-09: `node test/dockerization-contract.test.js` covered the UI test-container Dockerfile, `.dockerignore`, no-runtime scope, and D1 docs; `docker build --target test -t groupscout-ui-test .`, `docker run --rm groupscout-ui-test`, and `npm test` passed.

Phase 12 D2 run on 2026-05-09: `node --test test/dockerization-contract.test.js` covered the browser runtime contract metadata, reserved start command, port `3000`, `/healthz`, static asset boundary, same-origin `/api/*` routing to `http://groupscout:8080`, forbidden browser public config keys, and D2 docs; `npm test` passed.

Phase 12 D3 run on 2026-05-09: `node --test test/dockerization-contract.test.js` covered the `compose.dev.yml` backend Compose override, `groupscout-ui` service, `groupscout_net` attachment, host `${GROUPSCOUT_UI_HOST_PORT:-3001}` to container `3000` port mapping, `/healthz` health harness, `http://groupscout:8080` internal target metadata, no-secret Compose boundary, and D3 docs; `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`, `npm test`, `docker build --target test -t groupscout-ui-test .`, and `docker run --rm groupscout-ui-test` passed.

Phase 12 D4 run on 2026-05-09: `node --test test/dockerization-contract.test.js` covered the production same-origin Node server, `web/dist` root and static asset serving, server-side `/api/*` proxy request construction, public-config secret rejection, the `production` Docker target, and D4 docs; `npm test`, `docker build --target test -t groupscout-ui-test .`, `docker run --rm groupscout-ui-test`, and `docker build --target production -t groupscout-ui-production .` passed. Smoke checks cover `GET /healthz`, `GET /`, `GET /assets/app.js`, and `GET /api/system`.

Phase 12 D5 run on 2026-05-09: `node test/dockerization-contract.test.js` first failed because the Docker operations docs, CI notes, and troubleshooting entries were missing. The green docs/checklist run covered local UI tests, containerized UI tests, dev Compose startup/teardown, backend dependency expectations, required UI Docker env vars, CI hook order, and troubleshooting splits; `node --test test/dockerization-contract.test.js`, `npm test`, `docker build --target test -t groupscout-ui-test .`, `docker run --rm groupscout-ui-test`, `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`, and `docker build --target production -t groupscout-ui-production .` passed. Production container smoke checks for `GET /healthz`, `GET /`, and `GET /assets/app.js` passed on host port `3006`; backend-dependent `GET /api/system` was not counted because `GET http://localhost:8080/health` could not connect.

Phase 13 run on 2026-05-09: `node test/phase-13-renderer-runtime.test.js` first failed because Phase 13 contract/renderer/dev-server/backend-smoke modules were missing, `npm run build` was absent, `compose.dev.yml` still used the D3 health harness, and D4 did not have app-route fallback. The green run covered the vanilla DOM renderer contract, dependency-free rendered route harness, Today/Leads/Lead Detail route rendering, static build output, public asset secret scan, D4 route fallback, product dev server Compose command, and backend compatibility smoke classification; `node --test test/phase-13-renderer-runtime.test.js`, `node --test test/dockerization-contract.test.js`, `npm run build`, and `npm test` passed.

Canonical Phase 0-15 run on 2026-05-09: `node test/baseline-reconciliation.test.js` first failed because the UI baseline files were locally deleted, then passed after restoring the tracked Phase 13 baseline. Focused red/green additions covered `/api/leads/{id}/raw` in backend compatibility smoke, `createApiClient().getLead(...)` for `GET /api/leads/{id}`, and Phase 15 deterministic browser UX hardening. `node --test test/baseline-reconciliation.test.js test/api-boundary.test.js test/lead-inbox-client.test.js test/phase-13-renderer-runtime.test.js test/browser-ux-hardening.test.js`, `npm run build`, `npm test`, Docker test-image build/run, production image build, and production-container smoke checks for `/healthz`, `/`, and `/assets/app.js` passed.

Production session-gate refresh on 2026-05-09: `node --test test/session-deployment.test.js` first failed because `web/src/server/productionServer.js` did not expose a request handler that authorized `/api/*` before proxying. It passed after `createProductionRequestHandler(...)` rejected missing/invalid `groupscout_session` when `UI_SESSION_SECRET` is configured, forwarded backend Docker smoke requests when session auth is unconfigured, forwarded valid sessions to the backend target, and applied CSP, `x-content-type-options`, `x-frame-options`, and `referrer-policy` headers.

Phase 15 renderer-evidence refresh on 2026-05-09: `node --test test/browser-ux-hardening.test.js` first failed because route-specific focus labels and rendered responsive modes were not reported for every primary route. It passed after the route shell accepted viewport options and the dependency-free renderer emitted controls/actions and desktop/tablet/mobile modes for Today, Leads, Lead Detail, Verification, Outreach, Pipeline, Analytics, and Alerts.

Phase 1 token refresh on 2026-05-09: `node --test test/design-tokens.test.js` first failed because `text-input-focused` was missing from the exported token map and `property-row` background drifted from `DESIGN.md`. It passed after the token export matched the documented component contracts.

Optional design-doc lint. This is not an npm script and may use the network through `npx`:

```sh
npx @google/design.md lint DESIGN.md
```

The D1 Docker target runs the current UI test suite in a clean Node container:

```sh
docker build --target test -t groupscout-ui-test .
docker run --rm groupscout-ui-test
```

The D2 browser runtime contract reserved a lightweight Node server with `npm run start:ui`, container port `3000`, and `/healthz`. D3 added the initial development Compose health harness. D4 implements production static serving and same-origin `/api/*` proxying. Phase 13 adds the dependency-free product renderer, static build, product dev server, and backend compatibility classifier. See [Phase 12 UI Dockerization Prompt Pack](./phase-12-ui-dockerization.md), [UI Dockerization Contract](./ui-dockerization-contract.md), and [Phase 13 Product Renderer Runtime](./phase-13-product-renderer-runtime.md).

Development Compose config validation from the UI source repo:

```sh
docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet
```

`compose.dev.yml` must be merged with the backend Compose file. It is intentionally not standalone because backend service `groupscout` and network `groupscout_net` come from `/mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml`.

Development Compose startup and teardown:

```sh
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout
curl -i http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/healthz
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml down
```

The Phase 13 UI service is `groupscout-ui`. It joins the backend `groupscout_net`, targets `http://groupscout:8080` server-side, exposes container port `3000` on host `${GROUPSCOUT_UI_HOST_PORT:-3001}`, serves `web/dist`, and healthchecks `/healthz`. A targeted smoke run should start `groupscout-ui` with backend service `groupscout`; the current backend dependency chain also starts `postgres`, `ollama`, and `ollama-init`.

Use `-p groupscout` when the D3 run will be followed by a D4 production-container smoke that attaches to `groupscout_groupscout_net`.

Production UI runtime smoke commands from the UI source repo:

```sh
docker build --target production -t groupscout-ui-production .
docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
```

Split these checks by dependency:

- Backend-independent UI runtime checks: `GET /healthz`, `GET /`, and `GET /assets/app.js`.
- Backend-dependent proxy checks: `GET /api/leads?limit=1`, `GET /api/system`, and `GET /api/alerts?limit=1` should reach the backend and return `200` when `UI_SESSION_SECRET` is not configured for Docker smoke. When `UI_SESSION_SECRET` is configured, unauthenticated `/api/*` requests should return `401` from the UI session gate.

Backend plus UI Docker smoke run on 2026-05-08:

```sh
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up -d --build groupscout-ui groupscout
curl -i http://localhost:8080/health
curl -i http://localhost:3001/healthz
docker build --target production -t groupscout-ui-production .
docker run --rm -d --name groupscout-ui-production-smoke --network groupscout_groupscout_net -p 3002:3000 -e UI_API_PROXY_TARGET=http://groupscout:8080 groupscout-ui-production
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/leads?limit=1
curl -i http://localhost:3002/api/system
curl -i http://localhost:3002/api/alerts?limit=1
```

Expected results: backend `/health` returns `200`; D3 UI `/healthz` returns `200`; D4 `/healthz`, `/`, and `/assets/app.js` return `200`; D4 `/api/leads?limit=1`, `/api/system`, and `/api/alerts?limit=1` reach the backend and return `200`. Treat `502` as Docker/proxy reachability failure.

Docker operations docs check:

```sh
node --test test/dockerization-contract.test.js
```

## CI Notes

CI should run `npm test`, build and run the D1 test image, then build the production image. If the backend Compose file is available, CI can also run the merged Compose config validation command. Smoke `/healthz`, `/`, and `/assets/app.js` against the production UI container without backend secrets; smoke `/api/system` only when the backend service or a CI stub is reachable.

Do not run CI UI containers with backend `.env` or `--env-file`. Do not inject `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` into browser-visible config, static assets, Compose output, or CI artifacts.

## Focused UI Tests

```sh
node --test test/api-boundary.test.js
node --test test/app-shell.test.js
node --test test/session-deployment.test.js
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
node --test test/dockerization-contract.test.js
node --test test/phase-13-renderer-runtime.test.js
node --test test/baseline-reconciliation.test.js
node --test test/browser-ux-hardening.test.js
```

API-client focused run for smell phases:

```sh
node --test test/api-boundary.test.js test/lead-inbox-client.test.js test/lead-status-mutation-client.test.js test/raw-audit-client.test.js test/outreach-client.test.js test/pipeline-client.test.js test/stats-client.test.js test/alert-client.test.js test/system-client.test.js
```

## What The Current UI Tests Cover

- Same-origin `/api/*` browser request boundary, public API-client facade shape, split adapter module ownership, request defaults, and invalid-route pre-fetch rejection.
- Session-cookie enforcement metadata for UI `/api/*` access.
- Production request-handler enforcement that rejects missing/invalid `groupscout_session` before `/api/*` proxying when `UI_SESSION_SECRET` is configured, allows the no-secret backend Docker smoke proxy path, and applies baseline browser security headers.
- Admin setup-token login route rendering, protected-route redirect metadata, login verification through auth status/current-admin calls, and logout routing through `/api/auth/logout`.
- Admin login floating-window renderer contract through the `admin-login-window` form class.
- `UI_ENABLED`, `UI_BASE_PATH`, `UI_SESSION_SECRET`, and development-only `CORS_ALLOWED_ORIGINS` deployment behavior.
- D2 browser runtime contract metadata for the reserved start command, port, health path, static asset boundary, `/api/*` server/proxy target, and forbidden browser public config keys.
- D3/Phase 13 development Compose metadata for the UI service, backend network attachment, backend service dependency, port mapping, healthcheck command, no-secret Compose boundary, and product dev-server health payload.
- D4 production same-origin server metadata, static asset presence, app-route fallback, server-side `/api/*` proxy request construction, public-config secret rejection, and production Docker target.
- Phase 13 renderer/runtime contract, rendered route smoke for Today/Leads/Lead Detail/Pipeline/mobile Verification, static product build output, app-route-only SPA navigation, public asset secret scans, copied `/src/*` module cache policy, product dev server metadata, and backend compatibility smoke classification.
- Phase 15 deterministic browser UX hardening for primary navigation, main landmarks, route-specific focus labels, accessible-name metadata, rendered desktop/tablet/mobile modes, stable loading/error/empty states, text-containment policy, and same-origin API metadata.
- Recursive browser-source checks that `API_TOKEN` is not referenced in browser-facing `web/src/**/*.js` modules outside `web/src/server`.
- Lead inbox query serialization, blank-filter elision, sort overrides, and response adaptation.
- Lead detail client access through same-origin `GET /api/leads/{id}`, encoded lead IDs, evidence workspace fields, source evidence, AI enrichment, reviewer corrections, and activity rows.
- Lead inbox mocked table, filters, states, responsive metadata, and accessibility metadata.
- Lead detail sections, source evidence, raw audit link intent, AI enrichment metadata, corrections, timeline, and states.
- Verification Queue desktop/tablet table models, mobile card rendering, raw audit links, and review actions.
- Lead status transition rules, invalid transition blocking, validation, and PATCH mutation intent.
- Verification queue trigger classification, filters, row actions, raw audit alias links, responsive metadata, and defined redaction-policy metadata for future inline previews.
- Raw audit client access through same-origin `GET /api/leads/{id}/raw`, encoded lead IDs, and raw policy defaults.
- Outreach client access through same-origin `GET/POST /api/leads/{id}/outreach`, encoded lead IDs, optional draft logging, and manual logging validation.
- Outreach workspace editable drafts, contact validation, manual copied/sent/logged states, outcome capture, activity timelines, responsive metadata, and route mounting.
- Pipeline client access through same-origin `GET/POST /api/pipeline/runs`, async run metadata, default manual run payloads, and compact health defaults.
- Pipeline monitor compact health fields, async run creation state, collector counts/failures, LLM health, notification delivery health, partial-data/error states, responsive metadata, and route mounting.
- Stats client access through same-origin `GET /api/stats`, full summary adaptation, and source-yield hit-rate policy metadata.
- Analytics dashboard status/source/score/owner/week summaries, source-yield hit-rate definition, lead aging, verification quality, upcoming demand, denominator/date-range labels, responsive metadata, and route mounting.
- Alert client access through read-only same-origin `GET /api/alerts`, cursor routing, and required evidence/inventory/history sections.
- Alertd console current alert state, SPS summaries, evidence, room inventory, action history, disabled mutation actions, Slack-first policy metadata, responsive metadata, and route mounting.
- System client access through read-only same-origin `GET /api/system` and required generated/pipeline/count sections.
- Today command center summary counts, priority lead rows, aging claimed work, active alerts, failed jobs, system health, read-only action policy, responsive metadata, and root route mounting.
- Selected design token exports used by the current screen models.

## What The Current UI Tests Do Not Cover

- Real browser-engine rendering.
- Real DOM focus behavior.
- Real keyboard navigation.
- Visual regression.
- CSS layout.
- Pixel proof that the admin login card floats correctly across viewport sizes.
- Live backend compatibility.
- Real cookie signing, browser session issuance, reverse-proxy behavior, and production CORS headers.
- A production CLI session store or login flow for creating valid `groupscout_session` cookies.
- Product browser runtime container smoke tests beyond the D4 health/static/proxy path and Phase 13 dependency-free rendered HTML smoke.
- Raw audit payload redaction behavior beyond the explicit blocked TODO.
- Real email sending, clipboard behavior, or CRM sync for outreach.
- Real pipeline execution, worker polling, Grafana rendering, or log viewer integration.
- Live analytics aggregation accuracy against backend storage.
- Custom dashboard/chart rendering behavior.
- Live alertd contract compatibility, Slack delivery behavior, and alert acknowledgement/resolution workflows.
- Live `/api/system` compatibility or real system-health aggregation accuracy.
- Full `DESIGN.md` token-reference resolution.
- The D4 `npm run start:ui` command is a production static/proxy server. The generated app asset is intentionally minimal until later browser-framework phases expand the renderer.

Before treating a UI feature as production-ready, add real browser or component-level coverage once a renderer framework or browser harness is introduced.

## Interpreting API Boundary Failures

- Absolute `http` or `https` browser requests fail before fetch because UI calls must stay same-origin.
- Non-`/api/` paths fail before fetch because `web/src/api/transport.js` owns the browser API transport boundary behind the `web/src/api/client.js` facade.
- Non-2xx responses throw `Request failed with status N`.
- API adapter errors usually mean the backend response shape drifted from the model contract. Check required sections such as `leads`, `date_range`, `summaries`, `pipeline`, `counts`, `alerts`, `evidence`, `room_inventory`, and `action_history`.
- Session/deployment failures usually come from a short `UI_SESSION_SECRET`, a missing `groupscout_session` when session auth is configured, or production `CORS_ALLOWED_ORIGINS` configuration.

## Backend Tests

Backend tests run from:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
```

Main backend suite:

```sh
make test
```

Equivalent command:

```sh
go test -v ./...
```

Package example:

```sh
go test -v ./internal/enrichment/...
```

Alertd tests:

```sh
go test ./cmd/alertd/... ./internal/alert/...
```

Postgres integration tests:

```sh
TEST_POSTGRES_URL="postgres://groupscout:groupscout@localhost:5432/groupscout?sslmode=disable" \
  go test -v -tags integration ./internal/storage/...
```

EvalOps:

```sh
make eval-quality
make eval-gate
make eval-target
```

## Manual API Checks

With the backend running on `localhost:8080`:

```sh
curl -i http://localhost:8080/health
```

```sh
curl -i -X POST \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  http://localhost:8080/run
```

After a Docker-triggered run:

```sh
docker compose logs groupscout --tail=50
```
