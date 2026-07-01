# Backend And Frontend Docker E2E

Use this runbook when you want the current GroupScout backend and the separate UI Docker image running together on one Docker network.

This is a smoke path, not a full product UI acceptance test. The backend is live, and the UI production container serves static assets and forwards `/api/*`.

Reconciled 2026-06-13 (`groupscout-site-crz`): `make smoke-ui-docker-e2e`, `scripts/smoke-ui-docker-e2e.sh`, and `internal/smoke` are now present. The planned UI API routes are still tracked by `groupscout-site-eqm`.

Docker remains the production and CI command baseline for this E2E path. Local Podman smoke gates passed on 2026-07-01; use [PODMAN_MIGRATION.md](../../guides/PODMAN_MIGRATION.md) for validated Podman commands, project/network naming, host aliases, and Windows port notes.

Executable form from the backend repo:

```sh
make smoke-ui-docker-e2e
```

## Repos

| Repo | Path |
|---|---|
| Backend | `/mnt/c/Users/alvin/GolandProjects/groupscout` |
| UI | `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui` |

## What Runs

| Runtime | Port | Purpose |
|---|---:|---|
| Backend `groupscout` service | `8080` | Live Go API container from backend Compose. |
| UI D3 product dev server | `3001` by default | Proves the UI service can build, join the backend Compose network, serve product assets, and expose `/healthz`. |
| UI D4 production container | `3002` in this runbook | Serves `web/dist`, exposes `/healthz`, and proxies same-origin `/api/*` to `http://groupscout:8080`. |
| UI D4 bad-proxy container | `3003` in this runbook | Uses an intentionally invalid upstream so `/api/*` must return `502`. |
| Grafana | `3000` | Backend observability stack, so the UI examples avoid host port `3000`. |

The D4 production UI runtime is wired through the UI repo's `smoke-ui-e2e` Compose profile. The backend-owned repeatable gate is tracked by `groupscout-site-crz`.

## Prerequisites

- Docker Desktop or Docker Engine with Compose v2.
- Backend `.env` in `/mnt/c/Users/alvin/GolandProjects/groupscout`.
- UI repo present at `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.
- Time for first backend boot: the current `groupscout` service depends on `ollama` and `ollama-init`, so first startup may pull models.

## Start Backend Plus D3 UI Product Server

Run from the UI repo so `compose.dev.yml` is local, but pin the Compose project name to `groupscout` so the Docker network name is stable:

For Podman Compose, keep the same `-p groupscout` convention but verify the generated network name before relying on manual container attachment.

```sh
cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui

docker compose -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  up -d --build groupscout groupscout-ui
```

Smoke the backend and the D3 UI product server:

```sh
curl -i http://localhost:8080/health
curl -i http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/healthz
```

Expected:

- Backend `/health` returns `200`.
- UI D3 `/healthz` returns `200`.

Use the D4 production profile on port `3002` for same-origin production proxy smoke checks.

## Start The D4 Production UI Profile

Still from the UI repo:

```sh
docker compose -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  --profile smoke-ui-e2e \
  up -d --build groupscout groupscout-ui-production groupscout-ui-production-bad-proxy
```

Smoke the frontend origin:

```sh
curl -i http://localhost:3002/healthz
curl -i http://localhost:3002/
curl -i http://localhost:3002/assets/app.js
curl -i http://localhost:3002/api/system
curl -i http://localhost:3003/api/system
```

Expected:

- `/healthz`, `/`, and `/assets/app.js` should return `200` from the UI container.
- `/api/system` on port `3002` should return the current backend status through the same-origin UI proxy: `404` on backend `main`, or `401` once protected UI API routes are present.
- `/api/system` on port `3003` should return `502`; this proves proxy failures are surfaced by the UI runtime instead of being mistaken for backend route drift.

The executable gate performs these checks and verifies the UI Compose overlay does not inject `API_TOKEN`, provider keys, Slack/email keys, database URLs, Ollama endpoints, or `UI_SESSION_SECRET`.

## Cleanup

```sh
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

Use `--volumes` only when you intentionally want to delete Postgres, n8n, Grafana, and Ollama volumes.

## Important Boundaries

- Do not run `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui/compose.dev.yml` by itself. It depends on backend service `groupscout` and network `groupscout_net`.
- Do not pass the backend `.env` into UI containers.
- Do not expose `API_TOKEN`, provider keys, Slack tokens, email-provider keys, database URLs, Ollama endpoints, or `UI_SESSION_SECRET` to browser-visible UI config or static assets.
- Browser-facing UI code should use relative `/api/*`; only the server-side D4 container should know `http://groupscout:8080`.

