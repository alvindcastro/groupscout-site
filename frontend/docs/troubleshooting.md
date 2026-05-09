# Troubleshooting

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
- Browser `/api/*` requests lack the `groupscout_session` cookie or present an invalid session value.

## Browser Runtime Contract Fails

Check `web/src/server/browserRuntimeContract.js` behavior through `test/dockerization-contract.test.js`.

Common causes:

- The D2 contract was changed away from the lightweight Node server model that D4 now implements for production static serving and server-side `/api/*` proxying.
- The reserved UI port, health path, or same-origin `/api/*` routing target no longer matches `docs/ui-dockerization-contract.md`.
- Browser public config includes automation credentials, provider keys, database URLs, or `UI_SESSION_SECRET`.

Note: D2 originally reserved `npm run start:ui`, port `3000`, and `/healthz` as contract metadata. D4 now provides the runnable production static/proxy server for those values. Phase 13 adds the dependency-free vanilla DOM renderer, static build, and product dev server; a browser framework remains future work.

## UI Development Compose Fails

D3 adds `compose.dev.yml` as a UI-repo override for the backend Compose file. Validate the merged config from the UI repo:

```sh
docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet
```

Do not validate or start `compose.dev.yml` by itself. It depends on backend-defined service `groupscout` and network `groupscout_net`.

Use `-p groupscout` when you need the generated network name to be `groupscout_groupscout_net` for the D4 production smoke container:

```sh
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout
```

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

Do not pass the backend `.env` file into UI containers with `--env-file`. UI Docker operations should not receive `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET`.

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

## UI Proxy Smoke Fails

Interpret `/api/*` smoke failures by status:

- `502`: the backend is unreachable or `UI_API_PROXY_TARGET` points at the wrong host.
- `404`: the proxy reached the backend, but the backend does not implement that `/api/*` route.
- `401`: `UI_SESSION_SECRET` is configured and the request does not have a valid `groupscout_session`; unset `UI_SESSION_SECRET` only for backend Docker smoke checks that need no-login proxy reachability.
- DNS errors for `groupscout`: the production container is not on the backend Compose network, or it is running with standalone `docker run` and should use `http://host.docker.internal:8080`.

Browser-visible code and config should still show relative `/api/*`, never `http://groupscout:8080` or backend secrets.

## Backend And UI Docker Mode Mismatch

D3/Phase 13 and D4 are different modes:

- Phase 13 `compose.dev.yml` runs `node web/src/server/productDevServer.js`; it serves `web/dist`, healthchecks `/healthz`, and keeps backend discovery server-side.
- D4 `groupscout-ui-production` runs `npm run start:ui`; it serves `web/dist` and proxies `/api/*`.

If `http://localhost:3001/api/system` fails with backend route drift, classify it the same way as D4 proxy smoke: `502` is proxy/backend reachability, `404` is live backend route drift, `401`/`403` is auth, and a `200` with missing fields is schema drift.

If D4 is run with standalone `docker run`, use a host backend target:

```sh
docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
```

If D4 is run on the backend Compose network, use the backend service name:

```sh
docker run --rm -d --name groupscout-ui-production-smoke --network groupscout_groupscout_net -p 3002:3000 -e UI_API_PROXY_TARGET=http://groupscout:8080 groupscout-ui-production
```

The current backend smoke contract expects `/api/leads?limit=1` to return `200` through the D4 proxy. `/api/system` may return backend `200` or `404`; both distinguish a reachable backend from `502` proxy failure.

## Production UI Runtime Fails

D4 adds the production same-origin server:

```sh
npm run start:ui
```

For Docker:

```sh
docker build --target production -t groupscout-ui-production .
docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
```

Common causes:

- `GET /` or `GET /assets/app.js` returns 404 because `web/dist/index.html` or `web/dist/assets/app.js` is missing from the image or local checkout.
- `GET /leads/lead_hotel_001` returns 404 because app-route fallback to `index.html` regressed.
- `GET /api/*` returns 401 because `UI_SESSION_SECRET` is configured and the request does not have a valid `groupscout_session`.
- `GET /api/*` returns 502 because `UI_API_PROXY_TARGET` is wrong, the backend is down, or a Compose container cannot resolve `groupscout`.
- Browser code references `http://groupscout:8080` directly instead of a relative `/api/*` path.
- Public config or static assets include blocked names such as `API_TOKEN`, `CLAUDE_API_KEY`, Slack tokens, Resend/SendGrid keys, database URLs, or `UI_SESSION_SECRET`.

Smoke the runtime from one origin:

```sh
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
```

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

- Some docs reference `docs/API_TESTING.md`, but that file was not present during inspection.
- Some docs use `SENDGRID_API_KEY`, while `.env.example` and config use `RESEND_API_KEY`.
- Some Postgres examples reference a generated container name, while compose sets `container_name: groupscout_postgres`.

Prefer the current backend `.env.example`, `config/config.go`, `docker-compose.yml`, and `Makefile` when commands disagree.
