# Phase 12 UI Dockerization Prompt Pack

Phase D0 through D5 are complete. Future UI runtime or renderer work still requires strict TDD before adding a framework, browser renderer, or broader application runtime code.

## Sources Inspected

- `/mnt/c/Users/alvin/GolandProjects/groupscout/Dockerfile`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/DOCKER.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/TESTING.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/TDD_AI_QUALITY.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/API_CONFIG.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/planning/ui/README.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/planning/ui/UI_STRATEGY.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/planning/ui/BACKEND_FOR_UI_TESTING.md`
- `package.json`
- `README.md`
- `docs/developer-guide.md`
- `docs/testing.md`
- `docs/ui-dockerization-contract.md`

## Current Findings

- The backend repo already has a full Docker Compose stack with `groupscout` on `8080`, `alertd` on `8081`, Postgres, n8n, Prometheus, Grafana, Loki, Promtail, Ollama, and `ollama-init`.
- Backend containers communicate on `groupscout_net`; UI containers should reach the backend as `http://groupscout:8080` when running in the same Compose project or network.
- The UI repo has a D1 `Dockerfile` test target, `.dockerignore`, D3 `compose.dev.yml` backend Compose override, D4 production same-origin Node server, and D5 Docker operations/CI documentation. Phase 13 later added a dependency-free renderer, browser build script, and product UI dev server without adding a framework or lockfile.
- The UI repo is currently a model-level plain JavaScript workspace with Node's built-in `node:test` runner and `npm test`.
- Browser UI code must use same-origin `/api/*` contracts.
- `API_TOKEN` is reserved for automation clients and must not be exposed to browser JavaScript.

## Recommended Direction

The current UI test workspace and production same-origin static/proxy server are containerized. Next, add a product browser runtime or renderer only after its behavior is test-covered. This avoids inventing UI framework code around a product app that does not exist yet.

Preferred end state:

- A UI test image that runs `npm test` deterministically.
- A development Compose service for the UI once a dev server exists.
- A same-origin browser path for `/api/*` through the D4 UI production server.
- No browser access to automation tokens.
- Smoke checks that prove the UI development container starts on the backend network and production static/proxy checks can run without exposing browser-visible secrets.

## Global TDD Rules

- [ ] Read the relevant UI docs, backend Docker docs, and current tests before editing files.
- [ ] Write or update the smallest failing test before adding Docker or runtime behavior.
- [ ] Run the narrow command and record the expected red failure.
- [ ] Implement only enough Docker/runtime/config to pass the failing test.
- [ ] Rerun the narrow command and record the green result.
- [ ] Run `npm test` after UI source or contract changes.
- [ ] Run Docker validation commands only after a Docker file or Compose file exists.
- [ ] Keep `/api/*` browser calls same-origin.
- [ ] Keep `API_TOKEN` out of browser code, images, static assets, and generated config.
- [ ] Update Markdown docs only when behavior, commands, ports, or acceptance criteria change.

## Phase D0 - Dockerization Contract And Decision Record

Canonical D0 output: [UI Dockerization Contract](./ui-dockerization-contract.md).

### Prompt

```text
Strictly follow TDD. Do not add Docker files yet.

Goal: define the UI dockerization contract before implementation.

Context:
- The UI repo currently has no renderer, no build tool, no lockfile, and no Docker files.
- The backend Docker stack lives in /mnt/c/Users/alvin/GolandProjects/groupscout.
- Backend Compose exposes groupscout on 8080 and alertd on 8081.
- Browser API calls must stay behind same-origin /api/* paths.
- API_TOKEN must never enter browser JavaScript, static assets, or public container config.

TDD requirements:
1. Add or update documentation tests, guardrail tests, or lint-style tests that describe required Docker docs and security boundaries.
2. Confirm the new test fails because the contract is not documented or exported yet.
3. Add the minimum Markdown/process metadata needed to pass.
4. Rerun the narrow test and npm test.

Do not create Dockerfile, Compose, nginx, dev-server, or app-renderer code in this phase.
```

### Tasks

- [x] Capture the chosen dockerization path: test image first, browser runtime later.
- [x] Document backend service names, ports, and internal URLs.
- [x] Document same-origin `/api/*` and no-`API_TOKEN` browser constraints.
- [x] Add a testable guardrail for Docker docs if the repo has a docs test pattern.
- [x] Record red and green evidence.

### Acceptance Criteria

- [x] Future Docker work has a written contract.
- [x] Security boundaries are explicit before any image or Compose file exists.
- [x] The current pass remains documentation-only.

### Implementation Notes

- The D0 guardrail test lives in `test/dockerization-contract.test.js`.
- Red run: `node --test test/dockerization-contract.test.js` failed because the contract file and documentation links were not present.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.
- D0 did not add `Dockerfile`, `.dockerignore`, Compose, proxy, dev-server, renderer, or application runtime files.

## Phase D1 - UI Test Container

### Prompt

```text
Strictly follow TDD.

Goal: add a deterministic Docker target that runs the current UI test suite without introducing a browser runtime.

Context:
- package.json has only "test": "node --test".
- There is no package install requirement or lockfile yet.
- The first Docker target should prove the existing model-level workspace can run in a clean Node container.

TDD requirements:
1. Write a failing test or script assertion for required Docker test-image behavior.
2. Confirm it fails because Dockerfile/.dockerignore or the documented command does not exist yet.
3. Add the smallest Dockerfile/.dockerignore needed for `npm test`.
4. Rerun the narrow assertion.
5. Run `docker build` and the containerized `npm test`.
6. Run local `npm test`.

Do not add a dev server, production web server, reverse proxy, or backend Compose wiring in this phase.
```

### Tasks

- [x] Add failing coverage for Dockerfile existence and expected test command.
- [x] Add failing coverage for `.dockerignore` excluding `.git`, `node_modules`, logs, and local IDE files.
- [x] Add the minimal Node image target for `npm test`.
- [x] Verify no secret env vars are baked into the image.
- [x] Document `docker build` and containerized test commands.

### Acceptance Criteria

- [x] The UI test suite runs inside a container.
- [x] The image does not require backend services.
- [x] The image does not expose browser runtime behavior.

### Implementation Notes

#### D1 Evidence

- The D1 guardrail coverage lives in `test/dockerization-contract.test.js`.
- Added `Dockerfile` with a single `test` target that runs `npm test` in Node without an install step.
- Added `.dockerignore` for `.git`, `node_modules`, logs, IDE files, generated outputs, and local `.env` files.
- Red run: `node test/dockerization-contract.test.js` failed because `Dockerfile`, `.dockerignore`, and D1 command documentation did not exist yet.
- Green run: `node test/dockerization-contract.test.js`.
- Docker build: `docker build --target test -t groupscout-ui-test .`.
- Containerized test: `docker run --rm groupscout-ui-test`.
- Full-suite run: `npm test`.
- D1 did not add a dev server, production web server, reverse proxy, backend Compose wiring, exposed port, healthcheck, or browser runtime.

## Phase D2 - Browser Runtime Contract

### Prompt

```text
Strictly follow TDD.

Goal: choose and test the minimal browser runtime contract before adding a dev server or renderer.

Context:
- The UI is still model-level JavaScript.
- A real operator UI needs a browser-rendered app, but the framework/runtime is not selected in this repo yet.
- Same-origin /api/* calls and session-cookie behavior from Phase 9 must remain intact.

TDD requirements:
1. Write failing tests for the intended runtime contract: start command, port, health path, static asset boundary, and /api/* routing expectation.
2. Confirm the tests fail because no runtime contract exists.
3. Add only the smallest runtime metadata or placeholder needed to pass the contract tests.
4. Rerun focused tests and npm test.

Do not build product UI screens beyond the existing phase scope. Do not choose a framework unless the tests and docs require it.
```

### Tasks

- [x] Decide whether the UI runtime is static assets, a lightweight Node server, or backend-served assets.
- [x] Define the UI container port.
- [x] Define the health check path for the UI service.
- [x] Define how `/api/*` reaches `groupscout:8080` without exposing `API_TOKEN`.
- [x] Add tests for runtime config and forbidden browser env leaks.

### Acceptance Criteria

- [x] A future UI server has a tested contract before Docker Compose wiring.
- [x] `/api/*` remains same-origin from the browser perspective.
- [x] The selected runtime does not weaken Phase 9 auth/session boundaries.

### Implementation Notes

#### D2 Evidence

- The D2 runtime contract lives in `web/src/server/browserRuntimeContract.js`.
- Selected runtime contract: lightweight Node server, no framework selected, reserved `npm run start:ui`, container port `3000`, and health path `/healthz`.
- Static assets are contractually server-owned generated assets under `web/dist`; generated public config remains disabled in D2.
- Future server/proxy-side `/api/*` traffic targets `http://groupscout:8080`, while browser code keeps same-origin `/api/*` and `credentials: "same-origin"` with the `groupscout_session` cookie.
- Red run: `node --test test/dockerization-contract.test.js` failed because the runtime contract module and D2 documentation did not exist.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.
- D2 did not add a dev server, production web server, reverse proxy, renderer, framework, Compose wiring, package start script, exposed Docker port, or Docker healthcheck.

## Phase D3 - Development Compose Integration

### Prompt

```text
Strictly follow TDD.

Goal: add development Compose wiring for the UI service and backend stack.

Context:
- The backend Compose file already defines groupscout_net and service name groupscout.
- UI containers should call the backend by service name when inside Compose.
- Host development should continue to support local `npm test`.

TDD requirements:
1. Write failing validation for the Compose service name, network attachment, port mapping, healthcheck, and backend URL/proxy target.
2. Confirm the validation fails before adding Compose config.
3. Add the smallest Compose file or override needed for UI development.
4. Run `docker compose config`.
5. Start only the minimum services required for a smoke check.
6. Rerun focused tests and npm test.

Do not require Ollama, n8n, Grafana, or the full observability stack for basic UI container startup unless the smoke test explicitly needs them.
```

### Tasks

- [x] Decide whether UI Compose lives in this repo or as an override beside the backend Compose file.
- [x] Attach the UI service to the backend Docker network.
- [x] Point internal API/proxy traffic at `http://groupscout:8080`.
- [x] Add a healthcheck for the UI service.
- [x] Add `docker compose config` validation to docs or scripts.
- [x] Document which backend services are required for UI smoke tests.

### Acceptance Criteria

- [x] Developers can run the UI development container alongside the backend Compose stack.
- [x] Compose config validates without requiring secrets in the UI image.
- [x] The basic UI development health path does not require n8n, Grafana, Prometheus, Loki, Promtail, or the full lead pipeline to run.

### Implementation Notes

#### D3 Evidence

- The D3 Compose override lives in `compose.dev.yml` and is intended to be used with the backend Compose file: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml ...`.
- The UI service is `groupscout-ui`; it builds the existing D1 `test` target, runs `node web/src/server/devComposeHealthServer.js`, attaches to `groupscout_net`, and depends on backend service `groupscout`.
- The UI development container uses container port `3000`, host port `${GROUPSCOUT_UI_HOST_PORT:-3001}`, and healthcheck path `/healthz`.
- Internal future server/proxy traffic is configured as `UI_API_PROXY_TARGET=http://groupscout:8080`, while browser-facing API metadata remains `/api/*`.
- Minimum backend services for the D3 smoke path are `groupscout`, `postgres`, `ollama`, and `ollama-init` because the current backend Compose `groupscout` service depends on them. D3 does not require `alertd`, `n8n`, `grafana`, `prometheus`, `loki`, or `promtail` for the UI health harness.
- Red run: `node test/dockerization-contract.test.js` failed because `compose.dev.yml`, `web/src/server/devComposeHealthServer.js`, and D3 documentation did not exist yet.
- Green run: `node --test test/dockerization-contract.test.js`.
- Docker Compose config: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`.
- Full-suite run: `npm test`.
- Docker build: `docker build --target test -t groupscout-ui-test .`.
- Containerized test: `docker run --rm groupscout-ui-test`.
- D3 did not add production same-origin proxying, static asset serving, a browser framework, a renderer, generated public config, or an implemented `npm run start:ui` package script.

## Phase D4 - Same-Origin Proxy Or Static Serving

### Prompt

```text
Strictly follow TDD.

Goal: implement the production same-origin serving model for browser assets and /api/*.

Context:
- Same-origin deployment is preferred.
- Browser code should call `/api/*`, not `http://groupscout:8080`.
- API_TOKEN remains server-side and automation-only.

TDD requirements:
1. Write failing tests for same-origin asset serving and /api/* forwarding behavior.
2. Write failing tests that production config rejects public automation-token injection.
3. Confirm expected red failures.
4. Add the smallest proxy/static-serving config needed.
5. Run focused tests, npm test, Docker build, and a smoke check against the running container.

Do not add role matrices, production identity-provider UI, or direct database access in this phase.
```

### Tasks

- [x] Choose proxy container versus backend-served static assets.
- [x] Test browser-visible config excludes `API_TOKEN`, `CLAUDE_API_KEY`, Slack tokens, Resend keys, and database URLs.
- [x] Test `/api/*` remains a relative browser path.
- [x] Add production container or backend static-asset serving config.
- [x] Add smoke checks for `/`, a static asset, and an `/api/*` route.

### Acceptance Criteria

- [x] Browser users see one origin for UI and `/api/*`.
- [x] Secrets stay server-side.
- [x] Production container behavior is covered by tests and smoke commands.

### Implementation Notes

#### D4 Evidence

- Chosen serving model: lightweight Node production server in the UI repo. It serves `web/dist` assets and forwards `/api/*` server-side to `http://groupscout:8080` by default.
- Added `npm run start:ui` for `node web/src/server/productionServer.js`.
- Added a production Docker target named `production` with container port `3000`, `/healthz` healthcheck, and no baked production secrets.
- Browser-visible config is whitelist-only and keeps `/api/*` relative. Public config/static-asset tests reject `API_TOKEN`, `CLAUDE_API_KEY`, Slack tokens, Resend/SendGrid keys, database URLs, provider keys, Ollama URLs, and `UI_SESSION_SECRET`.
- Red run: `node test/dockerization-contract.test.js` failed because `web/src/server/productionServer.js`, `web/dist`, the production Docker target, and D4 documentation did not exist.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.
- Docker build: `docker build --target production -t groupscout-ui-production .`.
- Smoke health: `GET /healthz`.
- Smoke root: `GET /`.
- Smoke static asset: `GET /assets/app.js`.
- Smoke API proxy: `GET /api/leads?limit=1`, `GET /api/system`, and `GET /api/alerts?limit=1` when a backend or CI stub is reachable.
- D4 did not add role matrices, production identity-provider UI, direct database access, a browser framework, or a product renderer.

## Phase D5 - Docker Operations Docs And CI Hooks

### Prompt

```text
Strictly follow TDD.

Goal: make UI Docker usage repeatable for local development and CI.

Context:
- Backend docs already cover Docker Compose for the full stack.
- UI docs should be explicit about what can be run without the full backend and what requires Compose.

TDD requirements:
1. Add failing docs/checklist coverage for the documented Docker commands.
2. Confirm expected failure before editing docs.
3. Update README, developer docs, testing docs, and troubleshooting docs with the minimum accurate commands.
4. Rerun docs checks, npm test, and Docker smoke checks.

Do not add new runtime behavior in this phase unless a failing operations test requires it.
```

### Tasks

- [x] Document local UI tests.
- [x] Document containerized UI tests.
- [x] Document dev Compose startup and teardown.
- [x] Document backend dependency expectations and required env vars.
- [x] Document troubleshooting for port conflicts, missing Docker Desktop/WSL integration, and proxy failures.
- [x] Add CI job notes for container build and smoke tests.

### Acceptance Criteria

- [x] A new developer can run local tests, container tests, and dev Compose from docs.
- [x] CI has a clear future path for UI image build and smoke checks.
- [x] Troubleshooting docs distinguish UI container failures from backend stack failures.

### Implementation Notes

#### D5 Evidence

- D5 changed documentation and guardrail coverage only; it did not change the Dockerfile, Compose file, production server, static assets, browser runtime contract, or product renderer.
- Documented local tests: `npm test`.
- Documented containerized tests: `docker build --target test -t groupscout-ui-test .` and `docker run --rm groupscout-ui-test`.
- Docker Compose config: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml config --quiet`.
- Docker Compose startup: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up --build groupscout-ui groupscout`.
- Docker Compose teardown: `docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml down`.
- Backend dependency notes: D3 development smoke starts `groupscout-ui` with backend service `groupscout`; backend Compose also starts `postgres`, `ollama`, and `ollama-init`. UI health does not require `alertd`, `n8n`, Grafana, Prometheus, Loki, Promtail, or a pipeline run.
- Required UI Docker env vars: `GROUPSCOUT_UI_HOST_PORT`, optional `GROUPSCOUT_UI_REPO`, and server-only `UI_API_PROXY_TARGET`.
- CI hook order: `npm test`, Docker test-image build/run, production image build, then optional smoke checks for `/healthz`, `/`, `/assets/app.js`, and backend-dependent `/api/leads?limit=1`, `/api/system`, and `/api/alerts?limit=1` when a backend or CI stub is reachable.
- D5 docs explicitly prohibit running UI containers with backend `.env` or `--env-file`, and prohibit injecting `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET` into browser-visible config, static assets, Compose output, or CI artifacts.
- Red run: `node test/dockerization-contract.test.js` failed because the D5 operations docs, CI notes, and troubleshooting entries did not exist.
- Green run: `node --test test/dockerization-contract.test.js`.
- Full-suite run: `npm test`.
- Docker build: `docker build --target test -t groupscout-ui-test .`.
- Containerized test: `docker run --rm groupscout-ui-test`.
- Docker production build: `docker build --target production -t groupscout-ui-production .`.
- Production smoke checks: `GET /healthz`, `GET /`, and `GET /assets/app.js` against the UI production container passed on host port `3006`.
- Backend-dependent `/api/leads?limit=1`, `/api/system`, and `/api/alerts?limit=1` smoke was not counted because `GET http://localhost:8080/health` could not connect; it requires a reachable backend or CI stub.

## Suggested Parallel Agent Prompts

Use these only when explicitly running parallel agents for a future implementation pass.

### Agent A - Backend Docker Compatibility

```text
Inspect /mnt/c/Users/alvin/GolandProjects/groupscout Docker and docs only. Do not edit files. Return the exact service names, ports, networks, healthchecks, env vars, and smoke-test endpoints the UI Docker plan must respect. Call out any Compose changes that would be risky.
```

### Agent B - UI Runtime And Test Contract

```text
Inspect /mnt/c/Users/alvin/WebstormProjects/groupscout-ui package.json, tests, web/src, and docs only. Do not edit files. Return the smallest TDD path for a UI Docker test image and the gaps that block a browser runtime image.
```

### Agent C - Security Boundary Review

```text
Inspect browser-facing UI modules and deployment/session docs. Do not edit files. Identify every place Docker, Compose, generated config, or static assets could accidentally expose API_TOKEN, provider keys, Slack tokens, Resend keys, or database URLs. Propose tests first.
```

### Agent D - Docs Integration

```text
Inspect README.md, docs/developer-guide.md, docs/testing.md, docs/troubleshooting.md, and UI_TDD_PHASE_PROMPTS.md. Do not edit files. Recommend the smallest Markdown updates needed after each Dockerization phase, keeping all task lists tickable and strict-TDD aligned.
```

## Resolved Decisions And Remaining Brainstorm

D2-D5 resolved the original Dockerization decisions: the runtime model is a lightweight Node server, development Compose lives in this UI repo as `compose.dev.yml`, production same-origin behavior uses the D4 Node static/proxy server, the UI healthcheck route is `/healthz`, CI runs `npm test` before Docker image checks, and Dockerized UI tests stay backend-independent by default.

Phase 13 answered the next runtime question with a dependency-free renderer, static build, product dev server, and compatibility smoke classifier while preserving the D0-D5 container, same-origin, and credential boundaries. See [Phase 13 Product Renderer Runtime](./phase-13-product-renderer-runtime.md).
