# Phase 38 Docker Runtime And End-To-End Smoke Contract

Phase 38 adds a backend-owned smoke script for the split backend plus external UI Docker runtime.

Reconciled 2026-06-13 (`groupscout-site-crz`): `scripts/smoke-ui-docker-e2e.sh`, `internal/smoke`, and the `make smoke-ui-docker-e2e` target are now present in the backend. The gate is runnable once both Compose stacks are up.

Script:

- `scripts/smoke-ui-docker-e2e.sh`

The script assumes the UI repo lives at `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui` unless `GROUPSCOUT_UI_REPO` overrides it.

## Checks

The smoke script verifies:

- Backend Compose health: `GET http://localhost:8080/health`.
- Production UI Compose profile validates with `docker compose ... --profile smoke-ui-e2e config --quiet`.
- Production UI container serves `/healthz`, `/`, and `/assets/app.js` on `${GROUPSCOUT_UI_PRODUCTION_HOST_PORT:-3002}`.
- Same-origin proxy reaches the backend: `GET /api/system` returns `404` on backend `main`, or `401` once protected UI API routes are present.
- A deliberately bad proxy target returns `502`, proving proxy failures are distinguishable from backend route misses.
- The UI Compose overlay does not contain `API_TOKEN`, database URLs, Slack/email/LLM keys, Ollama endpoints, or `UI_SESSION_SECRET`.
- Cleanup removes the production UI and bad-proxy Compose service containers without tearing down backend data volumes.

## Evidence

Red evidence:

- `go test ./internal/smoke` failed because `scripts/smoke-ui-docker-e2e.sh` did not exist.

Historical green evidence:

- `go test ./internal/smoke`
- `make smoke-ui-docker-e2e`

The gate is intentionally backend-owned because it verifies the backend Compose stack plus the external UI production runtime on one Docker network. The production UI services live in the UI repo's `smoke-ui-e2e` Compose profile.
