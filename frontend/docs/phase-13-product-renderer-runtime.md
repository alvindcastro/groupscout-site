# Phase 13 Product Renderer Runtime

Phase 13 introduces the first tested product renderer/runtime without adding framework dependencies. It keeps D4 as the production static/proxy boundary, adds a dependency-free vanilla DOM renderer smoke harness, generates static assets under `web/dist`, extends `compose.dev.yml` to run the product dev server, and records backend compatibility smoke classification for live `/api/*` route drift.

Detailed copy-paste prompts and tickable future tasks live in [Phase 13 Product Renderer Runtime Prompt Pack](./phase-13-product-renderer-runtime-prompts.md).

## Current Baseline

- The repo is still a plain JavaScript UI workspace tested with Node's built-in `node:test`.
- The D1 Docker `test` target runs `npm test` without installing dependencies or contacting backend services.
- The Phase 13 `compose.dev.yml` service is now a product dev server on `groupscout_net`; it still maps `${GROUPSCOUT_UI_HOST_PORT:-3001}` to container port `3000`.
- The D4 production server still runs with `npm run start:ui`, serves `web/dist`, exposes `/healthz`, forwards `/api/*` server-side to `UI_API_PROXY_TARGET` or `http://groupscout:8080`, and now falls back to `index.html` for app routes such as `/leads/{id}`.
- Static app navigation intercepts only first-party app routes. Same-origin `/api/*` links, `/assets/*`, copied `/src/*` modules, file-extension URLs, download links, target links, external links, and modified clicks keep normal browser behavior.
- Copied source modules under `web/dist/src` are served with `cache-control: no-store`; immutable caching remains limited to versioned static assets under `/assets/*`.
- Browser-facing API code must keep relative same-origin `/api/*` paths and `credentials: "same-origin"`.
- Automation credentials, provider keys, Slack tokens, Resend/SendGrid keys, database URLs, `OLLAMA_BASE_URL`, and `UI_SESSION_SECRET` must stay out of browser-visible config and static assets.

## Implemented Contract

- Renderer: `vanilla-dom`, implemented in `web/src/renderer/domRenderer.js`.
- Browser/component harness: dependency-free rendered HTML smoke tests in `test/phase-13-renderer-runtime.test.js`.
- Production serving: D4 Node static/proxy server remains the production boundary.
- Development serving: `compose.dev.yml` now starts `node web/src/server/productDevServer.js`.
- Build output: `npm run build` writes `web/dist/index.html`, `web/dist/assets/app.js`, `web/dist/assets/styles.css`, and copied dependency-free modules under `web/dist/src`.
- Public config: `UI_PUBLIC_API_PATH` is the only allowlisted public config key; secret-like names fail the Phase 13 guardrail.
- Backend compatibility: `web/src/server/backendCompatibilitySmoke.js` classifies proxy failure, backend route drift, auth requirements, schema drift, compatible responses, and backend errors.

## Evidence

- Red run: `node test/phase-13-renderer-runtime.test.js` failed because Phase 13 modules were missing, `npm run build` was absent, the Compose command still used the D3 health harness, and static route fallback was not implemented.
- Green run: `node --test test/phase-13-renderer-runtime.test.js` passed after adding the Phase 13 runtime contract, harness decision, minimal renderer, static build, product dev server, Compose update, backend smoke classifier, and D4 route fallback.
- Broader run: `npm test` covers Phase 13 plus the existing model/API/runtime guardrails.
- Review-fix run on 2026-05-09: `node --test test/phase-13-renderer-runtime.test.js test/dockerization-contract.test.js` covered raw-audit-safe click interception and the `/src/*` `no-store` cache policy.

## Runtime Options

### Static Build Plus D4 Node Server

Chosen for production. The product app compiles to `web/dist`, and D4 continues to serve static files plus server-side `/api/*` proxying.

Why it fits:

- Preserves one production browser origin.
- Keeps backend service names and secrets server-side.
- Fits the current Docker production target with the smallest conceptual change.

- Still no dependency lockfile is needed because the first renderer is dependency-free.
- Generated assets are scanned for forbidden public secrets.

### Lightweight Dev Server Plus Static Production Build

Chosen for development. Development uses the product dev server, while production remains the D4 static/proxy server.

Why it fits:

- Gives fast local UI iteration once real browser screens exist.
- Keeps production deployment independent from the dev server.

- Compose health still checks `/healthz`.
- Backend service discovery still uses `http://groupscout:8080` server-side.

### Backend-Served Product Assets

The backend serves built UI assets directly in production.

Why it may fit later:

- Can reduce one deployable artifact if the backend owns production serving.

Watchouts:

- Moves UI deployment coupling into the Go backend.
- Requires a stronger cross-repo contract and backend tests before replacing D4.

## Phase 13 Acceptance Criteria

- [x] A written renderer/runtime contract names the chosen dev and production serving model.
- [x] The first failing test proves the chosen contract before adding dependencies or framework code.
- [x] Browser-facing code still cannot call absolute backend URLs.
- [x] Generated public config remains whitelist-only and cannot include secret-like keys.
- [x] The production smoke path still covers `GET /healthz`, `GET /`, `GET /assets/app.js`, and one same-origin `/api/*` request when a backend or stub is reachable.
- [x] Documentation states that `compose.dev.yml` now runs a real product dev server.
- [x] README, developer guide, testing docs, troubleshooting docs, and changelog are updated when behavior changes.

## Prompt Pack Summary

- Phase 13-A: renderer/runtime contract before dependencies.
- Phase 13-B: browser and component test harness decision.
- Phase 13-C: minimal renderer mount for existing screen models.
- Phase 13-D: static build and public asset safety.
- Phase 13-E: product dev server and Compose behavior.
- Phase 13-F: live backend compatibility smoke classification.

## Suggested Parallel Agent Prompts

Use these only for a future implementation planning pass.

### Agent A - Renderer Fit

```text
Inspect /mnt/c/Users/alvin/WebstormProjects/groupscout-ui web/src/app, DESIGN.md, and tests only. Do not edit files. Recommend the smallest renderer approach that can mount the existing screen models, preserve design-token usage, and support browser-level tests.
```

### Agent B - Docker Runtime Impact

```text
Inspect Dockerfile, compose.dev.yml, package.json, web/src/server, and docs/ui-dockerization-contract.md only. Do not edit files. Identify the exact Docker and Compose changes a product dev server or static build would require, and which D0-D5 guarantees must stay unchanged.
```

### Agent C - Browser Test Strategy

```text
Inspect current node:test coverage and app screen models only. Do not edit files. Propose the first browser/component tests that should fail before adding a renderer, including routing, API boundary, accessibility, and responsive smoke coverage.
```

### Agent D - Security Boundary

```text
Inspect browser-facing web/src modules, web/src/server, and docs/troubleshooting.md only. Do not edit files. List how generated assets, public config, dev-server env injection, and Compose output could expose secrets, then propose guardrail tests first.
```

## Explicit Non-Goals For This Brainstorm

- Choosing a framework without failing tests.
- Replacing the D4 production static/proxy server.
- Moving production UI serving into the backend without a cross-repo contract.
- Passing backend `.env` or automation tokens into UI containers.
- Treating metadata-only accessibility and responsive tests as equivalent to browser behavior.
