# Troubleshooting

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/frontend/docs/troubleshooting.md`. Run UI diagnostics from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`; run backend diagnostics from `/mnt/c/Users/alvin/GolandProjects/groupscout`.

## Operator Smoke Checklist

Use this first when a user asks whether the UI is usable.

```sh
curl -i --max-time 5 http://localhost:8080/health
curl -i --max-time 5 http://localhost:3001/healthz
curl -i --max-time 5 http://localhost:3001/
```

Expected:

- Backend `/health` is HTTP `200` and includes `"database":"ok"`.
- UI `/healthz` is HTTP `200`.
- UI `/` is HTTP `200` and includes `/assets/app.js`.

Check app routes:

```sh
for path in / /leads /leads/lead_hotel_001 /verification /outreach /pipeline /analytics /alerts; do
  curl -i --max-time 5 "http://localhost:3001$path" | head -n 1
done
```

For the inspected current checkout, `/admin/login` is intentionally not part of this loop because the admin route is planned but absent. Expected: HTTP `200` for app-shell routes, or a login redirect when session auth is enabled after admin/session reconciliation lands. A `404` on app-shell routes means the route fallback, static build, or running container is stale; a `404` on `/admin/login` alone is expected in the current checkout.

Check API proxy routes:

```sh
curl -i --max-time 5 http://localhost:3001/api/leads?limit=1
curl -i --max-time 5 http://localhost:3001/api/system
curl -i --max-time 5 http://localhost:3001/api/alerts?limit=1
```

Interpret failures:

| Status | Meaning | First check |
|---|---|---|
| `502` | UI proxy cannot reach backend | `UI_API_PROXY_TARGET`, Docker network, backend container |
| `404` | Backend route missing or proxy path drifted | backend branch and `/api/*` route list |
| `401`/`403` | Session or backend auth is required | `/api/auth/status`, `groupscout_session`, `ADMIN_AUTH_ENABLED` |
| stale HTML or old screen | Browser or container is serving old assets | rebuild UI, hard-refresh, check `/assets/app.js` query |

## UI Tests Fail Immediately

Confirm the Node runtime is modern enough:

```sh
node --version
```

Use Node `18+` or newer. The test suite uses built-in `node:test` and modern web API objects.

## Browser API Boundary Test Fails

The UI intentionally blocks browser-facing code from hard-coding external API URLs or credentials.

Check:

- New app files under `web/src` should route API calls through `createApiClient(...)` from `web/src/api/client.js`; feature-specific API logic belongs in the focused modules under `web/src/api/`.
- Same-origin paths must start with `/api/`.
- The credential guard in `test/api-boundary.test.js` recursively scans browser-facing `web/src/**/*.js` files and skips `web/src/server`.
- Browser API calls intentionally use `credentials: "same-origin"` and do not inject `Authorization`, `x-api-key`, or `x-api-token` headers.

Common causes:

- A browser module calls an absolute `http` or `https` URL.
- A browser module calls a backend path outside `/api/`.
- A new browser-facing file was placed under `web/src/server`, which is treated as server-only by the credential scan.

## API Adapter Test Fails

The client adapters intentionally fail loudly when backend response shapes drift from the UI contract.

Check the required response sections named in the error. Common required sections include:

- `leads` for `GET /api/leads`.
- `lead`, `source_url`, `source_name`, `raw_audit_path`, `collected_at`, `ai`, `reviewer_corrections`, and `activity` for `GET /api/leads/{id}`.
- `date_range`, `denominator`, `summaries`, `source_yield`, `verification_quality`, and `demand` for `GET /api/stats`.
- `alerts`, `evidence`, `room_inventory`, and `action_history` for `GET /api/alerts`.
- `generated_at`, `health`, `pipeline`, and `counts` for `GET /api/system`.

Non-2xx fetch responses throw `Request failed with status N` before adapter logic runs.

## Lead Inbox Data Looks Missing In Tests

Current lead data is mocked in `web/src/app/leadInbox.js`. Filtering is model-level and in-memory.

Common causes:

- `minScore` is higher than the mocked row score.
- `owner: "unowned"` only returns rows with no owner.
- Date filters compare the `YYYY-MM-DD` slice of `createdAt`.
- `verificationState`, `source`, `status`, and `property` must match the mocked values exactly.

## Lead Detail Shows Not Found

Current detail data is mocked in `web/src/app/leadDetail.js`.

The shell extracts `/leads/{id}` using a raw string slice. Query strings, encoded IDs, and trailing slashes are not normalized yet because the route shell is still a phase mock.

When real routing is introduced, normalize and decode route params before lookup.

## Status Action Is Hidden Or Rejected

Allowed status actions come from `LEAD_STATUS_TRANSITIONS` in `web/src/app/leadStatus.js`.

If an action is missing:

- Confirm the lead status is one of the v1 statuses.
- Confirm the action is allowed from that status.
- Confirm required fields are present, such as `owner`, `notes`, `snoozeUntil`, `corrections`, or `correctionReason`.

Note: `follow_up` and `corrected` are actions, not statuses. They preserve the current status.

## UI Deployment Readiness Fails

Check `web/src/server/uiDeployment.js` behavior through `test/session-deployment.test.js`.

Common causes:

- `UI_ENABLED` is true but `UI_SESSION_SECRET` is missing or shorter than 32 characters.
- `CORS_ALLOWED_ORIGINS` is configured for a production environment. CORS allow lists are development-only in the current deployment model.
- Helper-level session authorization fails because browser `/api/*` requests lack the `groupscout_session` cookie or present an invalid session value. The current production server applies this helper before proxying when `UI_SESSION_SECRET` is configured.

## Browser Runtime Contract Fails

Check `web/src/server/browserRuntimeContract.js` behavior through `test/dockerization-contract.test.js`.

Common causes:

- The D2 contract was changed away from the lightweight Node server model that D4 now implements for production static serving and server-side `/api/*` proxying.
- The reserved UI port, health path, or same-origin `/api/*` routing target no longer matches `docs/ui-dockerization-contract.md`.
- Browser public config includes automation credentials, provider keys, database URLs, or `UI_SESSION_SECRET`.

Note: D2 originally reserved `npm run start:ui`, port `3000`, and `/healthz` as contract metadata. D4 now provides the runnable production static/proxy server for those values. Phase 13 adds the dependency-free vanilla DOM renderer, static build, and product dev server; real-browser harness work remains tracked by `groupscout-site-kb4`.

## UI Development Compose Fails

D3 adds `compose.dev.yml` as a UI-repo override for the backend Compose file. Use [Docker Runtime Matrix](./docker-runtime-matrix.md) for the current validation, startup, smoke, and teardown commands.

For Podman migration checks, use [Podman Migration Runbook](../../backend/docs/guides/PODMAN_MIGRATION.md). Keep `-p groupscout` during validation and verify the generated network name before relying on manual network attachment.

Do not validate or start `compose.dev.yml` by itself. It depends on backend-defined service `groupscout` and network `groupscout_net`.

Inspect service state and UI logs:

```sh
docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml ps
docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml logs groupscout-ui --tail=100
```

Common causes:

- The backend Compose file path is wrong or the sibling backend repo is not present at `/mnt/c/Users/alvin/GolandProjects/groupscout`.
- The override was run by itself instead of being merged with `/mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml`.
- Docker Desktop or WSL integration is not running, so `docker compose` cannot inspect or build services.
- Host port `3001` is already in use. Set `GROUPSCOUT_UI_HOST_PORT` to another host port; the container still listens on `3000`.
- The UI service cannot resolve `groupscout` because the override was run without the backend Compose file or without the `groupscout_net` network definition.
- The `groupscout` backend service is not started. A targeted D3 smoke run should include `groupscout-ui` and `groupscout`; backend Compose also starts `postgres`, `ollama`, and `ollama-init` because `groupscout` depends on them.

Phase 13 healthchecks `/healthz` on the product dev server. It serves the generated `web/dist` assets and preserves server-side `http://groupscout:8080` backend discovery.

## Docker Operations Fail Before Startup

If `docker compose config` cannot connect to the daemon, start Docker Desktop, verify WSL integration for the active distro, and rerun:

```sh
docker version
docker compose version
```

If testing Podman, check `podman info` and `podman compose version` instead. Podman does not use Docker Desktop WSL integration or the Docker socket permission model.

Do not pass the backend `.env` file into UI containers with `--env-file`. `UI_SESSION_SECRET` is allowed only as server-side runtime configuration for real operator deployments. UI container operations should not put `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` in browser-visible config, static assets, Compose output, or CI artifacts.

## Testing Stack Looks Partially Unhealthy

For ordinary UI/API smoke, these checks are enough before deeper debugging:

```sh
curl -i --max-time 5 http://localhost:8080/health
curl -i --max-time 5 http://localhost:3001/healthz
curl -i --max-time 5 http://localhost:3001/
```

Acceptable smoke state:

- Backend `/health` is `200`.
- Backend health JSON includes `"database":"ok"`.
- UI `/healthz` is `200`.
- UI `/` returns the generated app shell.

Known caveat: backend health may include `"ollama":"unavailable"` while `groupscout_ollama` is container-healthy. That is not a blocker for UI rendering, API proxy, admin-login, or Postgres-backed contract testing. It is a blocker for claiming LLM enrichment, model, or pipeline-quality behavior is healthy.

## Docker Port Conflicts

The backend stack publishes Grafana on host port `3000`, so the UI development Compose service defaults to host port `3001`.

Use another development host port:

```sh
GROUPSCOUT_UI_HOST_PORT=3005 docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout
```

Use another standalone production smoke host port:

```sh
docker run --rm -p 3005:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
```

For Podman standalone smoke, the host backend alias is usually `host.containers.internal`:

```sh
podman run --rm -p 3005:3000 -e UI_API_PROXY_TARGET=http://host.containers.internal:8080 groupscout-ui-production
```

## UI Proxy Smoke Fails

Interpret `/api/*` smoke failures by status:

- `502`: the backend is unreachable or `UI_API_PROXY_TARGET` points at the wrong host.
- `404`: the proxy reached the backend, but the requested backend route is missing or the path is wrong. On the current backend source snapshot, `/api/leads?limit=1`, `/api/system`, and `/api/alerts?limit=1` may return `404` until `groupscout-site-eqm` lands the UI API routes.
- `401`: expected only once protected UI API routes and session auth are present and the request does not have a valid `groupscout_session`; unset `UI_SESSION_SECRET` only for backend Docker smoke checks that need no-login proxy reachability.
- DNS errors for `groupscout`: the production container is not on the backend Compose network, or it is running standalone and should use a host alias such as Docker Desktop's `http://host.docker.internal:8080` or Podman's `http://host.containers.internal:8080`.

Browser-visible code and config should still show relative `/api/*`, never `http://groupscout:8080` or backend secrets.

## Planned Admin Login Redirects Unexpectedly

After the planned admin/session flow lands, protected app routes call `/api/auth/status` before rendering. If a route redirects to `/admin/login`, check:

- The backend is reachable through the UI proxy.
- `GET /api/auth/status` returns `auth_required: true` and `authenticated: true` after login.
- The browser received the HttpOnly `groupscout_session` cookie from `POST /api/auth/login`.
- The setup token was read from the current `ADMIN_SETUP_TOKEN` value or `ADMIN_SETUP_TOKEN_FILE`. File-backed setup tokens rotate after successful login.

When `UI_SESSION_SECRET` is configured, `/api/*` requests without a valid cookie return `401` before proxying. If `UI_SESSION_SECRET` is unset for container smoke, API requests can proxy without a browser session, but the backend may still require its own admin auth depending on `ADMIN_AUTH_ENABLED`.

## Planned Admin Login Does Not Appear In Docker

In the inspected current checkout, `/admin/login` is absent and that absence is not stale-asset evidence. After the planned admin route exists, first verify the running containers are not stale using [Admin Token Flow](./admin-token-flow.md), which owns the auth-status, token, logout, and stale asset recovery commands for that flow.

Expected auth status:

```json
{"auth_required":true,"authenticated":false,"setup_token_file":"data/admin-setup-token"}
```

Expected HTML asset version:

```html
<script type="module" src="/assets/app.js?v=admin-login-2"></script>
```

After the planned auth endpoints exist, if auth status is `404`, the backend container is old and must be rebuilt from the admin-login branch. If `/` still references an older static asset query such as `pipeline-output-4`, the UI container is old or the static build was not regenerated. If the HTML references the expected admin-login asset query but the browser still shows the old app, hard-refresh the browser or open `http://localhost:3001/admin/login` directly.

## Planned Admin Login Is Still Full Width

After the planned admin login route exists, `/admin/login` should render the token form as a compact floating window. If it still appears as a wide embedded panel, use [Admin Token Flow](./admin-token-flow.md) for rebuild and stale asset recovery commands.

Check that `web/src/renderer/domRenderer.js` renders `class="admin-login-window"` on the login form and that `web/src/renderer/staticStyles.css` makes `.admin-login` a transparent centering stage while `.admin-login-window` owns the border, background, and shadow. If source is correct but Docker is stale, rebuild the UI service and hard-refresh the browser.

## Setup Token Is Rejected

Check the active token source with the commands in [Admin Token Flow](./admin-token-flow.md).

The setup-token file is `/app/data/admin-setup-token` inside the backend container. [Admin Token Flow](./admin-token-flow.md) shows the current `docker exec` command for reading it from the container's `/app` working directory.

Common causes:

- File-backed setup tokens rotate after successful login. Re-read `/app/data/admin-setup-token` with `docker exec groupscout_app sh -lc 'cat data/admin-setup-token'`.
- `ADMIN_SETUP_TOKEN` is set in the backend environment, so the file-backed token is ignored.
- The backend restarted, which clears in-memory sessions and requires another login.
- The session expired after `ADMIN_SESSION_TTL_HOURS`, default `24`.

For direct API smoke, keep cookies in a jar as shown in [Admin Token Flow](./admin-token-flow.md).

## Logout Does Not End Session

After the planned admin auth flow lands, the logout control posts `/api/auth/logout`, which revokes the backend session when a token is present and expires `groupscout_session`. If a user remains authenticated, verify that the UI proxy forwards `/api/auth/logout` as a public auth endpoint and that the backend branch includes session revocation support.

## Backend And UI Docker Mode Mismatch

D3/Phase 13 and D4 are different modes:

- Phase 13 `compose.dev.yml` runs `node web/src/server/productDevServer.js`; it serves `web/dist`, healthchecks `/healthz`, and keeps backend discovery server-side.
- D4 `groupscout-ui-production` runs `npm run start:ui`; it serves `web/dist` and proxies `/api/*`.

If `http://localhost:3001/api/system` fails with backend route drift, classify it the same way as D4 proxy smoke: `502` is proxy/backend reachability, `404` is missing backend route or wrong path, `401`/`403` is auth, and a `200` with missing fields is schema drift.

If D4 is run with standalone `docker run`, use a host backend target. If D4 is run on the backend Compose network, use the backend service name. The exact commands live in [Docker Runtime Matrix](./docker-runtime-matrix.md).

The current backend smoke contract expects `/api/system` to return the backend's current status through the D4 proxy. On backend `main`, that is `404`; once protected UI API routes are present, expect `401` without a browser session. A `502` means the UI proxy could not reach the backend target.

The production UI Compose profile is implemented in the UI source repo. The backend-owned repeatable UI Docker E2E gate is available as `make smoke-ui-docker-e2e` in the backend source repo.

## Production UI Runtime Fails

D4 adds the production same-origin server:

```sh
npm run start:ui
```

For Docker startup and smoke commands, use [Docker Runtime Matrix](./docker-runtime-matrix.md).

Common causes:

- `GET /` or `GET /assets/app.js` returns 404 because `web/dist/index.html` or `web/dist/assets/app.js` is missing from the image or local checkout.
- `GET /leads/lead_hotel_001` returns 404 because app-route fallback to `index.html` regressed.
- `GET /api/*` returns 401 because `UI_SESSION_SECRET` is configured and the request does not have a valid `groupscout_session`.
- `GET /api/*` returns 502 because `UI_API_PROXY_TARGET` is wrong, the backend is down, or a Compose container cannot resolve `groupscout`.
- Browser code references `http://groupscout:8080` directly instead of a relative `/api/*` path.
- Public config or static assets include blocked names such as `API_TOKEN`, `CLAUDE_API_KEY`, Slack tokens, Resend/SendGrid keys, database URLs, or `UI_SESSION_SECRET`.

Smoke the runtime from one origin with the checks listed in [Docker Runtime Matrix](./docker-runtime-matrix.md).

## Backend Health Check Fails

Run backend commands from:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
```

Check server health:

```sh
curl -i http://localhost:8080/health
```

For Docker:

```sh
docker compose ps
docker compose logs groupscout --tail=50
docker compose logs postgres --tail=50
```

If the backend health status is `200` but only the `ollama` field is unavailable, inspect the model service separately:

```sh
docker compose ps ollama
docker compose logs ollama --tail=50
docker exec groupscout_ollama ollama list
```

Some backend docs still mention `docker compose logs app`; the current compose service is `groupscout`.

## Backend Returns Few Or No Leads

The backend filters data at multiple stages:

- Collector-level relevance filters.
- Database deduplication.
- Pre-scoring and `ENRICHMENT_THRESHOLD`.

Useful backend knobs:

- `MIN_PERMIT_VALUE_CAD`
- `ENRICHMENT_THRESHOLD`
- `PRIORITY_ALERT_THRESHOLD`

Inspect backend logs after a run:

```sh
docker compose logs groupscout --tail=50
```

## Backend Environment Looks Inconsistent

Known backend documentation drift observed during housekeeping:

- Long-lived markdown docs are centralized in `/mnt/c/Users/alvin/groupscout-site/backend` and `/mnt/c/Users/alvin/groupscout-site/frontend`; the backend and frontend source repos may no longer contain those files.
- Some historical docs reference `docs/API_TESTING.md`, but that file was not present during inspection.
- Some historical docs use `SENDGRID_API_KEY`, while `.env.example` and config use `RESEND_API_KEY`.
- Some historical Postgres examples reference a generated container name, while compose sets `container_name: groupscout_postgres`.

Prefer the current backend source files `.env.example`, `config/config.go`, `docker-compose.yml`, and `Makefile` when commands disagree.
