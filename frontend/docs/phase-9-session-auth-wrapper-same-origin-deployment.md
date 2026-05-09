# Phase 9 Session/Auth Wrapper And Same-Origin Deployment

Phase 9 adds the minimum deployment and session model needed to keep the operator workspace behind browser sessions while preserving automation-only credentials for non-browser clients.

## Scope

- `web/src/server/uiDeployment.js` owns UI deployment config parsing, readiness validation, base-path mount resolution, development-only CORS metadata, and session-cookie API authorization.
- `createMountedRouteShell(...)` maps deployed URLs below `UI_BASE_PATH` back to internal workspace routes and emits base-path-aware navigation hrefs.
- Browser API access remains same-origin `/api/*` through `createApiClient(...)` with `credentials: "same-origin"`.
- Recursive browser-source tests scan `web/src` browser modules and exclude server-only deployment helpers from bundle credential checks.

## Deployment Settings

- `UI_ENABLED` defaults to enabled and can disable UI shell/API access when set to `false`, `0`, `off`, or `no`.
- `UI_BASE_PATH` defaults to `/` and normalizes values such as `ops` to `/ops`.
- `UI_SESSION_SECRET` is required when the UI is enabled and must be at least 32 characters.
- `CORS_ALLOWED_ORIGINS` is accepted only for development-mode CORS metadata. Production readiness fails if a CORS allow-list is configured.

## Session Boundary

- Browser access to `/api/*` requires the `groupscout_session` cookie to match a server-side session store when `UI_SESSION_SECRET` is configured.
- Missing or invalid sessions return `401` with a `GroupScoutSession` challenge when session auth is configured.
- The production request handler applies this authorization check before proxying `/api/*` to `UI_API_PROXY_TARGET`; with no session secret configured, the backend Docker smoke path can proxy `/api/*` without a browser login flow.
- Health, static, JSON, and proxied responses include baseline browser security headers: CSP, `x-content-type-options`, `x-frame-options`, and `referrer-policy`.
- Disabled UI access returns `404` so the operator UI is not mounted.
- `API_TOKEN` remains reserved for automation clients such as n8n and is not injected into browser requests.

## Design Tokens

Phase 9 does not add a new visual surface and does not require new `DESIGN.md` token names.

## TDD Evidence

- Red run: `node --test test/session-deployment.test.js test/api-boundary.test.js test/app-shell.test.js` failed because `web/src/server/uiDeployment.js` and `createMountedRouteShell(...)` did not exist.
- Targeted green run: `node --test test/session-deployment.test.js test/api-boundary.test.js test/app-shell.test.js`.
- Full-suite green run: `npm test`.
- Refresh red run on 2026-05-09: `node --test test/session-deployment.test.js` failed because the production request handler did not expose session-gated `/api/*` behavior.
- Refresh green run on 2026-05-09: `node --test test/session-deployment.test.js` passed after `createProductionRequestHandler(...)` gated `/api/*` with `authorizeUiApiRequest(...)` and added browser security headers.

## Out Of Scope

- Complex role matrices.
- Password, MFA, OAuth, or identity-provider UI.
- Replacing automation `API_TOKEN` clients.
- Production reverse-proxy configuration.
- A server framework or browser renderer.
