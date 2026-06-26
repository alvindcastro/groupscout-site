# UI Dockerization Contract

D0 status: documentation-only. This contract records the path for future Docker work before any image, Compose, proxy, dev server, renderer, or browser runtime is added.

D1 status: UI test container. The first Docker target now runs the current model-level `npm test` suite in a clean Node container without adding browser runtime behavior.

D2 status: browser runtime contract. The future browser runtime shape is now test-covered as contract metadata without adding a runnable server, framework, dev server, proxy, or Compose wiring.

D3 status: development Compose integration. The UI repo provides a backend Compose override for development container wiring. Phase 13 now runs a product dev server from that override while preserving the same network, port, health, and no-secret guarantees.

D4 status: production same-origin serving. The UI repo now provides a lightweight Node production server that serves `web/dist` assets and proxies `/api/*` server-side to the backend target from one browser origin.

D5 status: Docker operations docs and CI hooks. The UI repo now documents repeatable local, container, Compose, production smoke, and future CI commands without adding runtime behavior.

Podman migration status: Docker remains the known-good baseline for this contract. Use [Podman Migration Runbook](../../backend/docs/guides/PODMAN_MIGRATION.md) before substituting Podman commands, host aliases, or network assumptions.

## Decision

The chosen dockerization path is test image first, browser runtime later.

- D1 will add the first Docker target: a deterministic Node image that runs the current `npm test` suite.
- D2 defines the browser runtime contract before selecting or wiring a dev server, renderer, proxy, or static-asset serving model.
- D3 wires development Compose through a UI-repo override that is loaded beside the backend Compose file.
- D4 wires production same-origin serving through the lightweight Node server path defined in D2.
- D5 documents the Docker operations path and CI hooks after the real commands exist.
- No Dockerfile, Compose file, reverse proxy, dev server, renderer, or application runtime is added in D0.

## D1 Test Image

The D1 Dockerfile has one target named `test`.

```sh
docker build --target test -t groupscout-ui-test .
docker run --rm groupscout-ui-test
```

The target uses Node, copies the no-install test inputs, and defaults to `npm test`. It does not expose ports, declare healthchecks, start a dev server, run a proxy, or connect to backend services.

The D1 `.dockerignore` excludes VCS metadata, `node_modules`, logs, IDE files, generated outputs, and local `.env` files so the test-image context stays small and does not include local secrets.

## D2 Browser Runtime Contract

The D2 runtime contract lives in `web/src/server/browserRuntimeContract.js`.

- Runtime model: `lightweight-node-server`
- Framework: `not-selected`
- Reserved start command: `npm run start:ui`
- UI container port: `3000`
- Health path: `/healthz`
- Static asset boundary: generated server-owned assets under `web/dist`, with no generated public config in D2.
- Browser route to APIs: same-origin `/api/*` with `credentials: "same-origin"` and the `groupscout_session` cookie.
- API proxy target: `http://groupscout:8080` for future server/proxy-side routing.

No framework, dev server, renderer, Compose service, or runnable UI server is added in D2. D4 now implements the reserved `start:ui` command.

## D3 Development Compose Integration

The D3 Compose override lives in `compose.dev.yml`.

- Compose override: `compose.dev.yml`
- UI service: `groupscout-ui`
- Backend service dependency: `groupscout`
- Shared backend network: `groupscout_net`
- Backend internal target: `http://groupscout:8080`
- Browser API path metadata: `/api/*`
- Health path: `/healthz`
- Host port: `${GROUPSCOUT_UI_HOST_PORT:-3001}` maps to container port `3000`
- Container command: `node web/src/server/productDevServer.js`

The development service builds the existing D1 `test` target and overrides the command with the Phase 13 product dev server. It remains a development Compose integration point, not the production serving model.

Use the override beside the backend Compose file:

```sh
docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet
```

The basic D3 smoke path targets `groupscout-ui` and the minimum backend services needed by the current backend `groupscout` dependency chain: `groupscout`, `postgres`, `ollama`, and `ollama-init`. It does not require `alertd`, `n8n`, `grafana`, `prometheus`, `loki`, `promtail`, or a lead pipeline run.

D3 still does not add production same-origin proxying, static asset serving, or a product UI renderer.

## D4 Production Same-Origin Serving

The D4 production server lives in `web/src/server/productionServer.js`.

- Serving model: `node-static-assets-and-api-proxy`
- Production command: `npm run start:ui`
- Production Docker target: `production`
- UI container port: `3000`
- Health path: `/healthz`
- Static asset root: `web/dist`
- Browser route to APIs: same-origin `/api/*`
- Server-side API proxy target: `http://groupscout:8080` by default, overrideable with `UI_API_PROXY_TARGET`

Build the production image:

```sh
docker build --target production -t groupscout-ui-production .
```

Run it with an explicit server-side proxy target:

```sh
docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
```

For Podman standalone smoke, validate `host.containers.internal` locally and use that alias when the backend runs on the host:

```sh
podman run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.containers.internal:8080 groupscout-ui-production
```

Smoke checks:

```sh
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i "http://localhost:3002/api/leads?limit=1"
curl -i http://localhost:3002/api/system
curl -i "http://localhost:3002/api/alerts?limit=1"
```

Browser JavaScript still sees only relative `/api/*` paths. `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, and `UI_SESSION_SECRET` remain server-side and must not enter static assets or public config.

## D5 Operations Docs And CI Hooks

D5 added operations documentation only. Phase 13 later changed `compose.dev.yml`, the production server fallback behavior, static assets, and product renderer/runtime under separate red-first coverage.

- Local test command: `npm test`
- Containerized test image build: `docker build --target test -t groupscout-ui-test .`
- Containerized test command: `docker run --rm groupscout-ui-test`
- Compose config validation: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`
- Development Compose startup: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout`
- Development Compose teardown: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml down`
- Production image build: `docker build --target production -t groupscout-ui-production .`
- Production smoke run: `docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production`

Required UI Docker env vars stay small:

- `GROUPSCOUT_UI_HOST_PORT` optionally changes the D3 development host port from `3001`.
- `GROUPSCOUT_UI_REPO` optionally changes the build context used by `compose.dev.yml`.
- `UI_API_PROXY_TARGET` is server-side only. Use `http://groupscout:8080` inside Compose and `http://host.docker.internal:8080` for a standalone Docker production-container smoke against a host backend. Podman standalone smoke usually uses `http://host.containers.internal:8080`; validate that alias locally.

The development smoke path requires the sibling backend repo at `/mnt/c/Users/alvin/GolandProjects/groupscout`. Starting `groupscout` also starts backend dependencies `postgres`, `ollama`, and `ollama-init`; the UI product dev-server smoke does not require `alertd`, `n8n`, Grafana, Prometheus, Loki, Promtail, or a lead pipeline run.

For a concise comparison of the D1 test image, Phase 13 development product server, and D4 production server, see [Docker Runtime Matrix](./docker-runtime-matrix.md).

CI order: local Node tests, Docker test-image build/run, production image build, then optional smoke checks. CI can validate merged Compose config when the backend Compose file is available. Production `/api/leads?limit=1`, `/api/system`, and `/api/alerts?limit=1` smoke checks need a reachable backend or CI stub and should return `200`; `/healthz`, `/`, and `/assets/app.js` can run against the UI container alone.

`UI_SESSION_SECRET` is allowed only as server-side runtime configuration for real operator deployments. CI must not inject `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` into browser-visible config, static assets, Compose output, or CI artifacts.

## Backend Contract

When future UI containers run with the backend Compose stack, they must use backend service names and internal network URLs rather than browser-visible backend origins.

- Backend service: `groupscout`
- Backend host port: `8080`
- Backend internal URL: `http://groupscout:8080`
- Alert service: `alertd`
- Alert host port: `8081`
- Alert internal URL: `http://alertd:8081`
- Shared backend network: `groupscout_net`

Browser code must not call `http://groupscout:8080` or `http://alertd:8081` directly. Those URLs are for container-to-container server/proxy use in later phases.

## Browser Security Boundary

- Browser API calls stay same-origin through `/api/*`.
- Browser requests remain session-cookie based and must preserve `credentials: "same-origin"` behavior.
- `groupscout_session` remains the browser session cookie name for UI `/api/*` access when the UI is enabled.
- `API_TOKEN` remains reserved for automation clients and must not enter browser JavaScript, static assets, generated public config, or image-baked browser environment.
- Future container config must keep provider keys, Slack tokens, Resend keys, database URLs, and automation credentials out of browser-visible layers and generated assets.
- `CORS_ALLOWED_ORIGINS` stays development-only; production should serve browser assets and `/api/*` from one origin.

## D0 Evidence

- Red run: `node --test test/dockerization-contract.test.js` failed because the D0 contract and documentation links were not present.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.

## D1 Evidence

- Red run: `node test/dockerization-contract.test.js` failed because `Dockerfile`, `.dockerignore`, and D1 command documentation were not present.
- Green run: `node test/dockerization-contract.test.js`.
- Docker build: `docker build --target test -t groupscout-ui-test .`.
- Container test run: `docker run --rm groupscout-ui-test`.
- Full-suite run: `npm test`.

## D2 Evidence

- Red run: `node --test test/dockerization-contract.test.js` failed because `web/src/server/browserRuntimeContract.js` and D2 documentation did not exist.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.

## D3 Evidence

- Red run: `node test/dockerization-contract.test.js` failed because `compose.dev.yml`, `web/src/server/devComposeHealthServer.js`, and D3 documentation did not exist.
- Green run: `node --test test/dockerization-contract.test.js`.
- Docker Compose config: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`.
- Full-suite run: `npm test`.
- Docker build: `docker build --target test -t groupscout-ui-test .`.
- Containerized test: `docker run --rm groupscout-ui-test`.

## D4 Evidence

- Red run: `node test/dockerization-contract.test.js` failed because `web/src/server/productionServer.js`, `web/dist`, the production Docker target, and D4 documentation did not exist.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.
- Docker test build: `docker build --target test -t groupscout-ui-test .`.
- Containerized test: `docker run --rm groupscout-ui-test`.
- Docker production build: `docker build --target production -t groupscout-ui-production .`.
- Smoke checks: `GET /healthz`, `GET /`, `GET /assets/app.js`, and backend-dependent `GET /api/leads?limit=1`, `GET /api/system`, and `GET /api/alerts?limit=1`.

## D5 Evidence

- Red run: `node test/dockerization-contract.test.js` failed because the D5 operations docs, CI notes, and troubleshooting entries did not exist.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.
- Docker test build: `docker build --target test -t groupscout-ui-test .`.
- Containerized test: `docker run --rm groupscout-ui-test`.
- Docker Compose config: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`.
- Docker production build: `docker build --target production -t groupscout-ui-production .`.
- Production smoke checks: `GET /healthz`, `GET /`, and `GET /assets/app.js` against the UI production container passed on host port `3006`.
- Backend-dependent `/api/leads?limit=1`, `/api/system`, and `/api/alerts?limit=1` smoke was not counted because `GET http://localhost:8080/health` could not connect; it requires a reachable backend or CI stub.

## Out Of Scope

- Nginx or Caddy configuration.
- Browser framework, renderer, or dev server implementation.
- Role matrices, production identity-provider UI, and direct database access.
- D5 runtime, Dockerfile, Compose, static asset, and server behavior changes.
