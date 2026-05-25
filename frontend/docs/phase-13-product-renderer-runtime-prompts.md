# Phase 13 Product Renderer Runtime Prompt Pack

> Implementation record for Phase 13. Future runtime changes must still follow TDD: red test first, smallest implementation, green test, then refactor.

## Sources Inspected

- `/mnt/c/Users/alvin/GolandProjects/groupscout/Dockerfile`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/DOCKER.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/TDD_AI_QUALITY.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/SUBAGENTS.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/planning/ui/UI_TDD_PHASE_PLAN.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/planning/ui/README.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/prompts/PROMPTS_PHASE31_UI.md`
- `UI_TDD_PHASE_PROMPTS.md`
- `docs/phase-12-ui-dockerization.md`
- `docs/phase-13-product-renderer-runtime.md`
- `docs/docker-runtime-matrix.md`
- `docs/testing.md`
- `Dockerfile`
- `compose.dev.yml`

## Current Findings

- The backend Docker stack builds the Go API and `alertd`, publishes `groupscout` on `8080`, publishes `alertd` on `8081`, and runs Postgres, n8n, Prometheus, Grafana, Loki, Promtail, Ollama, and `ollama-init` on `groupscout_net`.
- Backend Compose service name for the API is `groupscout`; UI containers on the backend network should target `http://groupscout:8080` server-side.
- Grafana already uses host port `3000`, so UI host examples should keep using `3001` for the development harness and `3002` for production-container smoke unless a future contract changes that.
- The UI repo already had a D1 test image, D3 Compose health harness, D4 production static/proxy server, and D5 Docker operations docs before Phase 13; Phase 13 upgrades the Compose command to the product dev server under test.
- The UI repo now has a dependency-free vanilla DOM renderer, rendered HTML smoke coverage, product dev server, static build script, and backend compatibility classifier. It still has no framework, lockfile, or dependency install step.
- Browser-facing code must keep relative `/api/*` paths and must not expose `API_TOKEN`, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, or `UI_SESSION_SECRET`.
- Current uncommitted UI changes are limited to `.idea/*` line-ending churn; there are no uncommitted product UI source changes in this working tree.

## Global TDD Rules For Phase 13

- Read the relevant docs and current tests before changing files.
- Write the smallest failing test or failing docs-contract assertion first.
- Run the narrow command and confirm the expected red failure.
- Implement only enough code or documentation to pass.
- Rerun the narrow command and confirm green.
- Run `npm test` after UI source, runtime contract, or docs-guardrail changes.
- Run Docker validation only after Docker, Compose, or runtime behavior changes.
- Add dependencies and a lockfile only after a failing test proves they are required.
- Keep the D4 production static/proxy server as the default production boundary unless a later tested contract replaces it.
- Keep browser API calls same-origin and keep generated public config whitelist-only.
- Record red evidence, green evidence, broader command, files changed, and residual risk in the task summary.

## Parallel Agent Prompts

Use these read-only prompts before implementation work. The main agent owns integration and final decisions.

### Agent A - Renderer Fit

```text
Read-only task. Inspect /mnt/c/Users/alvin/WebstormProjects/groupscout-ui web/src/app, DESIGN.md, docs/phase-13-product-renderer-runtime.md, and test/*.test.js. Recommend the smallest renderer approach that can mount the existing screen models, preserve dense operator workflows, use DESIGN.md tokens, and support browser/component tests. Do not edit files. Return risks, required tests, and files likely affected.
```

### Agent B - Docker Runtime Impact

```text
Read-only task. Inspect /mnt/c/Users/alvin/WebstormProjects/groupscout-ui Dockerfile, compose.dev.yml, package.json, web/src/server, docs/ui-dockerization-contract.md, and /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml. Identify exact Docker, Compose, port, healthcheck, and smoke-test changes a real product dev server or static build would require. Do not edit files. Preserve D1-D5 guarantees in the recommendation.
```

### Agent C - Browser Test Strategy

```text
Read-only task. Inspect current node:test coverage, app screen models, design tokens, API client boundary, and server runtime contracts. Propose the first browser/component tests that should fail before adding a renderer, including route mounting, same-origin API calls, accessibility, responsive behavior, and static asset secret scans. Do not edit files.
```

### Agent D - Backend Compatibility

```text
Read-only task. Inspect /mnt/c/Users/alvin/GolandProjects/groupscout docs/planning/ui, docs/prompts, internal/api if present, docker-compose.yml, and UI docs for modeled /api/* routes. List current likely route drift between the UI client contracts and backend live routes. Do not edit files. Recommend strict-TDD backend or UI follow-up prompts for compatibility smoke.
```

## Phase 13-A - Renderer Runtime Contract

### Prompt

```text
You are working in the GroupScout UI repo. Strictly follow TDD.

Goal: create a tested renderer/runtime contract before choosing or installing a UI framework.

Context:
- The repo is currently model-level JavaScript with node:test.
- D4 already serves static assets from web/dist and proxies /api/* server-side.
- The backend Docker stack is in /mnt/c/Users/alvin/GolandProjects/groupscout and exposes service groupscout on the Docker network.
- Browser-facing code must keep same-origin /api/* requests.

TDD requirements:
1. Write failing tests or docs-contract assertions for the chosen dev serving model, production serving model, build output path, health path, and static asset boundary.
2. Write failing guardrails that public config remains whitelist-only and cannot expose secret-like keys.
3. Run the narrow test and confirm it fails because the Phase 13 contract is missing.
4. Add the smallest contract module and Markdown needed to pass.
5. Rerun the narrow test and npm test.

Do not install dependencies, add a framework, create a dev server, or change Docker/Compose in this phase.
```

### Tasks

- [x] Decide whether production remains D4 static/proxy for Phase 13.
- [x] Decide that development changes from D3 health-only to product dev server once renderer dev-server tests exist.
- [x] Define the build output path, default port, and health path.
- [x] Define generated public config allowlist behavior.
- [x] Add red-first contract coverage.
- [x] Add the minimum docs/metadata to pass.
- [x] Update `docs/phase-13-product-renderer-runtime.md` with the chosen contract.

### Acceptance Criteria

- [x] The renderer/runtime decision is explicit before dependencies are added.
- [x] D1-D5 Docker guarantees remain intact.
- [x] Browser-visible config cannot contain backend or provider secrets.

## Phase 13-B - Browser And Component Test Harness Decision

### Prompt

```text
Strictly follow TDD.

Goal: choose the smallest browser/component test strategy needed to verify real rendered behavior.

Context:
- Current tests verify screen models, not DOM rendering, focus behavior, computed CSS, or responsive layout.
- A renderer should not be trusted until browser-visible behavior has failing tests.

TDD requirements:
1. Write failing tests or docs-contract assertions that name the required harness capabilities before adding packages.
2. Cover route rendering, keyboard/focus behavior, accessible names, responsive navigation, same-origin API calls, and static asset secret scans.
3. Confirm the failure is caused by missing harness support.
4. Add only the minimum harness configuration needed by the first rendered smoke test.
5. Rerun the narrow command and npm test.

Do not broaden into visual-regression infrastructure until route and API-boundary smoke tests pass.
```

### Tasks

- [x] Choose the first browser/component harness by required capability, not preference.
- [x] Define the first rendered route smoke: `/`, `/leads`, and one `/leads/{id}` case.
- [x] Define accessibility assertions for labels, landmarks, focusable controls, and tab/row actions.
- [x] Define responsive assertions for left navigation and detail/evidence layout.
- [x] Define API-boundary assertions proving relative `/api/*` requests.
- [x] Add a dependency/lockfile only after the failing harness test requires it.

### Acceptance Criteria

- [x] Browser tests cover behavior current model tests cannot cover.
- [x] The first harness does not require live Slack, email, LLM, public websites, or paid APIs.
- [x] Static asset scans stay in place after a browser build exists.

## Phase 13-C - Minimal Renderer Mount

### Prompt

```text
Strictly follow TDD.

Goal: mount the existing screen models in a real browser renderer without changing product scope.

Context:
- Existing screen models already describe Today, Leads, Lead Detail, Verification, Outreach, Pipeline, Analytics, Alerts, and Settings placeholders.
- The first renderer should consume these models instead of rewriting business logic into components.

TDD requirements:
1. Write failing rendered tests for the smallest route set before adding component code.
2. Test that rendered components use the existing API client boundary and screen-model inputs.
3. Test loading, empty, and error states for at least one data screen.
4. Implement only the minimum renderer shell and component mapping needed to pass.
5. Rerun rendered tests, focused model tests, and npm test.

Do not introduce live backend calls, new workflow mutations, or CRM/email behavior in this phase.
```

### Tasks

- [x] Mount Today as the first route.
- [x] Mount Lead Inbox with fixture-backed data.
- [x] Mount Lead Detail with fixture-backed evidence.
- [x] Keep screen model factories separate from renderer components.
- [x] Verify component props do not contain backend secrets.
- [x] Keep `createApiClient(...)` as the only browser API entry point.

### Acceptance Criteria

- [x] Existing model-level behavior is preserved in rendered UI.
- [x] Operators can render and navigate the primary shell routes in a browser test.
- [x] No live backend dependency is introduced for component tests.

## Phase 13-D - Static Build And Asset Safety

### Prompt

```text
Strictly follow TDD.

Goal: add a static product build that D4 can serve from web/dist.

Context:
- D4 already serves web/dist and proxies /api/*.
- Static assets must not include automation credentials, backend secrets, or server-only config.

TDD requirements:
1. Write failing build-output tests for index.html, app asset presence, route fallback behavior, and public config allowlist.
2. Write failing asset scans for API_TOKEN, database URLs, Slack/email/LLM keys, OLLAMA_BASE_URL, and UI_SESSION_SECRET.
3. Confirm failures before adding the build command.
4. Add the smallest build script and output needed to pass.
5. Rerun build-output tests, npm test, and D4 static/proxy smoke checks when runtime behavior changes.

Do not replace the production server unless a tested contract explicitly does so.
```

### Tasks

- [x] Add red-first coverage for generated asset names and paths.
- [x] Add red-first coverage for route fallback from D4.
- [x] Add red-first public asset secret scan.
- [x] Add or update the build command only after tests fail.
- [x] Verify `GET /healthz`, `GET /`, and `GET /assets/app.js`.
- [x] Verify one `/api/*` smoke only when backend or stub is reachable.

### Acceptance Criteria

- [x] D4 can serve the real product build.
- [x] Public assets expose only approved config.
- [x] Backend-dependent smoke failures distinguish backend 404 from proxy 502.

## Phase 13-E - Product Dev Server And Compose

### Prompt

```text
Strictly follow TDD.

Goal: replace or extend the D3 health harness only when a real product dev server exists.

Context:
- Current compose.dev.yml is a backend Compose override and only serves /healthz.
- Backend service groupscout and network groupscout_net come from the backend Compose file.
- Grafana uses host port 3000, so keep GROUPSCOUT_UI_HOST_PORT defaulting to 3001 unless the contract changes.

TDD requirements:
1. Write failing Compose/runtime tests for service command, healthcheck, port mapping, backend network attachment, and server-side API target.
2. Confirm failures before changing compose.dev.yml.
3. Add the smallest Compose changes needed for the product dev server.
4. Run docker compose config with the backend Compose file.
5. Run the minimum smoke check and npm test.

Do not pass backend .env files or automation tokens into UI containers.
```

### Tasks

- [x] Decide whether D3 remains health-only or becomes product dev server mode.
- [x] Test host port `${GROUPSCOUT_UI_HOST_PORT:-3001}`.
- [x] Test internal target `http://groupscout:8080` remains server-side.
- [x] Test healthcheck path and expected JSON.
- [x] Test Compose output does not include browser-visible secrets.
- [x] Document startup and teardown commands.

### Acceptance Criteria

- [x] The UI service can run beside backend Compose without port conflict.
- [x] Backend service discovery uses Docker service names inside the network.
- [x] UI Compose remains an override, not a standalone backend replacement.

## Phase 13-F - Live Backend Compatibility Smoke

### Prompt

```text
Strictly follow TDD.

Goal: turn current UI-modeled /api/* route assumptions into tested backend compatibility smoke checks.

Context:
- Prior D4 smoke reached the backend but received 404 for /api/system and /api/leads.
- UI tests model same-origin routes such as /api/leads, /api/pipeline/runs, /api/stats, /api/alerts, and /api/system.
- Backend route drift should be handled deliberately, not hidden behind UI fixtures.

TDD requirements:
1. Write failing compatibility tests or smoke docs that distinguish proxy failure, backend 404, auth failure, and schema drift.
2. Confirm the current red state against live backend or a documented stub.
3. Implement the smallest backend or UI contract alignment in the owning repo in a later coding pass.
4. Rerun backend tests, UI tests, and backend-plus-UI smoke after implementation.

Do not change backend code during this planning-only pass.
```

### Tasks

- [x] Inventory live backend routes versus UI client contracts.
- [x] Decide which repo owns each drift fix.
- [x] Define smoke expectations for `GET /api/system`.
- [x] Define smoke expectations for `GET /api/leads`.
- [x] Define auth/session expectations separately from automation `API_TOKEN`.
- [x] Add follow-up prompts for backend TDD implementation only after route ownership is clear.

### Acceptance Criteria

- [x] Smoke checks identify route drift without mislabeling it as Docker failure.
- [x] Backend and UI docs agree on route ownership.
- [x] Future backend implementation prompts start with failing HTTP/storage tests.

## Explicit Non-Goals

- Do not add production UI code in this planning pass.
- Do not install dependencies in this planning pass.
- Do not choose a framework by preference alone.
- Do not replace D4 production serving without a tested contract.
- Do not expose automation credentials to browser-visible config or assets.
- Do not treat model-level tests as sufficient for DOM focus, layout, or accessibility once a renderer exists.
