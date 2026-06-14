# Docker Runtime Matrix

This doc distinguishes the current Docker/runtime modes so contributors do not expect the test image, development product server, and production static/proxy server to behave identically.

Reconciled 2026-06-13 (`groupscout-site-crz`): the backend now exposes `make smoke-ui-docker-e2e`. The remaining frontend admin login route is tracked by `groupscout-site-1x9`.

## Modes

| Mode | Entry Point | Serves Assets | Proxies `/api/*` | Backend Required | Primary Use |
| --- | --- | --- | --- | --- | --- |
| D1 test image | `docker run --rm groupscout-ui-test` | No | No | No | Run the model-level Node test suite in a clean container. |
| Phase 13 development product server | `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout` | Yes, from `web/dist` | Server-side target metadata and same-origin behavior | Yes, for merged Compose wiring | Prove the UI service can join the backend Compose network, expose `/healthz`, and serve generated product assets. |
| D4 production server | `npm run start:ui` or `docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production` | Yes, from `web/dist` | Yes, server-side and session-gated when UI session auth is configured | Backend required for `/api/*` smoke checks | Serve one browser origin for static assets plus same-origin API forwarding. |
| D4 production server on backend network | `docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml --profile smoke-ui-e2e up -d --build groupscout groupscout-ui-production groupscout-ui-production-bad-proxy` | Yes, from `web/dist` | Yes, server-side and session-gated when UI session auth is configured | Yes | Production UI runtime smoke path; backend-owned repeatable gate restoration is tracked by `groupscout-site-crz`. |

## D1 Test Image

The `test` Docker target uses `node:22-bookworm-slim`, copies the no-install test inputs, and runs `npm test`. It has no exposed port, healthcheck, backend network dependency, product dev server, static asset server, or proxy behavior.

Use it when validating model contracts and documentation guardrails:

```sh
docker build --target test -t groupscout-ui-test .
docker run --rm groupscout-ui-test
```

## D3 Development Compose Health Harness

`compose.dev.yml` is an override for the backend Compose stack, not a standalone Compose file. The backend Compose file defines service `groupscout` and network `groupscout_net`; without that file, Compose validation is expected to fail.

The Phase 13 service builds the D1 `test` target but overrides the command with `node web/src/server/productDevServer.js`. It serves generated `web/dist` assets, exposes `/healthz` JSON metadata, and keeps backend discovery server-side through `http://groupscout:8080`.

Use it when validating container networking beside the backend stack:

```sh
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout
curl -i http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/healthz
docker compose -p groupscout -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml down
```

The `-p groupscout` flag keeps the Docker network name predictable as `groupscout_groupscout_net`, which is useful when attaching the D4 production container manually.

## D4 Production Server

The `production` Docker target runs `npm run start:ui`, which starts `web/src/server/productionServer.js`. In the inspected UI checkout, it serves static assets from `web/dist`, exposes `/healthz`, authorizes `/api/*` through the UI session boundary, and forwards authorized API requests server-side to `UI_API_PROXY_TARGET` or `http://groupscout:8080`.

Use it when validating same-origin static/proxy behavior:

```sh
docker build --target production -t groupscout-ui-production .
docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
```

Unauthenticated `GET /api/system` returns `401` from the UI before proxying when `UI_SESSION_SECRET` is configured and no valid `groupscout_session` is present. Backend-network smoke paths that do not configure UI session auth can still use `/api/*` to distinguish backend `404` route absence from `502` proxy reachability failure. `GET /healthz`, `GET /`, and `GET /assets/app.js` can validate the UI container without a backend.

To smoke the production UI container against the backend container instead of a host backend, use the `smoke-ui-e2e` Compose profile:

```sh
docker compose -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  --profile smoke-ui-e2e \
  up -d --build groupscout groupscout-ui-production groupscout-ui-production-bad-proxy

curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
curl -i http://localhost:3003/api/system

docker compose -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  --profile smoke-ui-e2e \
  stop groupscout-ui-production groupscout-ui-production-bad-proxy
docker compose -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  --profile smoke-ui-e2e \
  rm -f groupscout-ui-production groupscout-ui-production-bad-proxy
```

The backend smoke contract expects `/healthz`, `/`, and `/assets/app.js` to return `200`, proxied `/api/system` to return the current backend status (`404` on backend `main`, or `401` once protected UI API routes are present), and the intentionally bad proxy target to return `502`.

Repeatable gate: run `make smoke-ui-docker-e2e` from the backend repo after `groupscout-site-crz` restores or reconciles that target.

## Stable Boundaries

- Browser-facing JavaScript should see relative `/api/*` paths, not `http://groupscout:8080`.
- `UI_API_PROXY_TARGET` is server-side only.
- Production `/api/*` proxying is session-gated before backend forwarding when `UI_SESSION_SECRET` is configured.
- The backend Docker smoke path leaves `UI_SESSION_SECRET` unset so proxy reachability can be checked without issuing a browser session.
- Do not pass backend `.env` files into UI containers.
- Do not inject `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` into browser-visible config or static assets.
- A future framework-backed product dev server should update this matrix before changing `compose.dev.yml` semantics again.
