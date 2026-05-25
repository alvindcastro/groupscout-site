# Docker Runtime Matrix

This doc distinguishes the current Docker/runtime modes so contributors do not expect the test image, development product server, and production static/proxy server to behave identically.

## Modes

| Mode | Entry Point | Serves Assets | Proxies `/api/*` | Backend Required | Primary Use |
| --- | --- | --- | --- | --- | --- |
| D1 test image | `docker run --rm groupscout-ui-test` | No | No | No | Run the model-level Node test suite in a clean container. |
| Phase 13 development product server | `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout` | Yes, from `web/dist` | Server-side target metadata and same-origin behavior | Yes, for merged Compose wiring | Prove the UI service can join the backend Compose network, expose `/healthz`, and serve generated product assets. |
| D4 production server | `npm run start:ui` or `docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production` | Yes, from `web/dist` | Yes, server-side; session-gated when `UI_SESSION_SECRET` is configured | Backend required for `/api/*` smoke checks | Serve one browser origin for static assets plus same-origin API forwarding. |
| D4 production server on backend network | `docker run --rm -d --name groupscout-ui-production-smoke --network groupscout_groupscout_net -p 3002:3000 -e UI_API_PROXY_TARGET=http://groupscout:8080 groupscout-ui-production` | Yes, from `web/dist` | Yes, server-side; session-gated when `UI_SESSION_SECRET` is configured | Yes | Current backend-owned Docker smoke path; there is no dedicated Compose lifecycle for this mode yet. |

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

The `production` Docker target runs `npm run start:ui`, which starts `web/src/server/productionServer.js`. It serves static assets from `web/dist`, exposes `/healthz`, adds baseline browser security headers, and forwards `/api/*` server-side to `UI_API_PROXY_TARGET` or `http://groupscout:8080`. If `UI_SESSION_SECRET` is configured, `/api/*` requires a valid `groupscout_session` before forwarding. If no session secret is configured, the proxy remains open for backend Docker smoke checks.

Use it when validating same-origin static/proxy behavior:

```sh
docker build --target production -t groupscout-ui-production .
docker run --rm -p 3002:3000 -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 groupscout-ui-production
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
```

Unauthenticated `GET /api/system` returns `401` from the UI before proxying when `UI_SESSION_SECRET` is configured. The backend-owned Docker smoke leaves `UI_SESSION_SECRET` unset, so `/api/leads?limit=1` and `/api/system` can prove proxy reachability without browser credentials. `GET /healthz`, `GET /`, and `GET /assets/app.js` can validate the UI container without a backend.

To smoke the production UI container against the backend container instead of a host backend, attach it to the backend Compose network:

```sh
docker build --target production -t groupscout-ui-production .
docker run --rm -d --name groupscout-ui-production-smoke --network groupscout_groupscout_net -p 3002:3000 -e UI_API_PROXY_TARGET=http://groupscout:8080 groupscout-ui-production
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/leads?limit=1
curl -i http://localhost:3002/api/system
curl -i http://localhost:3002/api/alerts?limit=1
docker stop groupscout-ui-production-smoke
```

This is the closest current backend plus frontend Docker path. It remains manual because `compose.dev.yml` runs the development product server, while D4 production is still a separate image target. The current backend smoke contract expects `/api/leads?limit=1`, `/api/system`, and `/api/alerts?limit=1` to return `200`, and an intentionally bad proxy target to return `502`.

## Stable Boundaries

- Browser-facing JavaScript should see relative `/api/*` paths, not `http://groupscout:8080`.
- `UI_API_PROXY_TARGET` is server-side only.
- Production `/api/*` proxying is session-gated before backend forwarding when `UI_SESSION_SECRET` is configured.
- The backend Docker smoke path leaves `UI_SESSION_SECRET` unset so proxy reachability can be checked without issuing a browser session.
- Do not pass backend `.env` files into UI containers.
- Do not inject `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` into browser-visible config or static assets.
- A future framework-backed product dev server should update this matrix before changing `compose.dev.yml` semantics again.
