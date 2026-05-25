# Nice To Knows

## Repo Boundaries

- UI repo: `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`
- Backend repo: `/mnt/c/Users/alvin/GolandProjects/groupscout`

The UI repo is intentionally lightweight right now. It models contracts and screen state before introducing a rendering framework.

## No Install Step Yet

`package.json` currently defines a no-install test command and the lightweight production UI server command:

```json
{
  "scripts": {
    "start:ui": "node web/src/server/productionServer.js",
    "test": "node --test"
  }
}
```

There is no lockfile, bundler, or framework dependency yet.

## Phase Docs Are The Product History

The phase docs in `docs/phase-*` explain what each UI slice added and what stayed out of scope.

Use them before changing scope:

- `docs/phase-0-product-contract.md`
- `docs/phase-1-lead-inbox-contract.md`
- `docs/phase-2-lead-inbox-ui.md`
- `docs/phase-3-lead-detail-evidence-workspace.md`
- `docs/phase-4-lead-status-actions-state-model.md`
- `docs/phase-5-verification-queue-raw-audit-review.md`
- `docs/phase-6-outreach-workspace-activity-log.md`
- `docs/phase-7-pipeline-monitor-run-controls.md`
- `docs/phase-8-basic-analytics-demand-signals.md`
- `docs/phase-9-session-auth-wrapper-same-origin-deployment.md`
- `docs/phase-10-later-alertd-read-only-console.md`
- `docs/phase-11-today-command-center-system-health.md`
- `docs/phase-12-ui-dockerization.md`
- `docs/ui-dockerization-contract.md`
- `docs/docker-runtime-matrix.md`
- `docs/phase-13-product-renderer-runtime.md`
- `docs/ui-tdd-phase-0-15-implementation.md`
- `docs/phase-15-browser-ux-hardening.md`

## Design Tokens Are Partial

`DESIGN.md` is larger than the exported token subset in `web/src/design/tokens.js`.

The tests currently assert the tokens needed by implemented phases. They do not prove every `DESIGN.md` reference is exported or resolvable.

## API Client Is The Browser Safety Boundary

Use `createApiClient(...)` for API access. It enforces same-origin `/api/*` paths and `credentials: "same-origin"`.

Avoid direct external backend URLs in browser-facing modules.

Admin login uses the same boundary. The browser submits the setup token to `/api/auth/login`, receives an HttpOnly `groupscout_session`, and never receives the automation `API_TOKEN`. File-backed setup tokens rotate after a successful login, so a rejected setup token often means the token file was already used once. Use [Admin Token Flow](./admin-token-flow.md) for current setup-token, login, logout, and stale asset recovery commands.

## Mocked Data Is Intentional

Current screen data is mocked so the UI can lock screen contracts before a renderer and live backend integration exist. This applies across Lead Inbox, Lead Detail, Verification, Outreach, Pipeline, Analytics, Alerts, and Today.

When live data is introduced, keep adapters at the boundary and avoid spreading raw backend field names throughout app modules.

## Design Doc Lint Is Optional

`DESIGN.md` documents this lint command:

```sh
npx @google/design.md lint DESIGN.md
```

It is not wired into `package.json` yet. Treat it as an optional design-doc check, especially when changing design-token language.

## UI Docs Mirror Some Backend Context

Backend startup notes in this repo are convenience notes for UI work, not the backend source of truth.

When backend examples disagree, prefer the backend repo's current `.env.example`, `config/config.go`, `docker-compose.yml`, and `Makefile`.

## Docker Modes Are Different

The UI repo has more than one Docker mode. Phase 13 `compose.dev.yml` runs a backend-network product dev server on `localhost:3001`; it serves generated `web/dist` assets and keeps backend discovery server-side. D4 `groupscout-ui-production` runs the production static/proxy server on container port `3000`; use the `smoke-ui-e2e` Compose profile to attach it to the backend network for backend plus UI smoke checks.

Use `docker compose -p groupscout` when combining backend and UI Compose files so service names and the shared network stay predictable.

There is now a dependency-free product renderer and a dedicated production Compose profile for the D4 runtime. On backend `main`, `/api/system` currently returns `404` through the D4 proxy; that still proves proxy reachability until the protected UI API routes land.

For current Docker mode commands and expected smoke results, use [Docker Runtime Matrix](./docker-runtime-matrix.md). If backend `/health` reports Ollama unavailable while the UI and Postgres-backed API checks pass, treat that as an LLM/enrichment-specific follow-up, not a blocker for UI/API smoke.

Repeatable gate: run `make smoke-ui-docker-e2e` from the backend repo.

## Known Refactor Follow-Up

Broader backend/frontend smell refactors from the missed-task audit are tracked by `groupscout-site-2h1`.

## Backend Docker Service Names

The current backend compose file names the API service `groupscout` and the container `groupscout_app`.

Prefer:

```sh
docker compose logs groupscout --tail=50
```

Use `docker compose ps` if documentation examples disagree.

## Useful Backend Make Targets

From `/mnt/c/Users/alvin/GolandProjects/groupscout`:

```sh
make help
make test
make run
make run-once
make docker-up
make docker-down
make docker-logs
make doctor
```

`make doctor` runs the backend environment health check.
