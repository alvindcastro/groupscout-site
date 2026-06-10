# Phase 9 Session/Auth Wrapper And Same-Origin Deployment

Phase 9 adds the minimum deployment and session model needed to keep the operator workspace behind browser sessions while preserving automation-only credentials for non-browser clients.

Status reconciliation, 2026-06-10: the inspected UI checkout has `uiDeployment.js` helper logic, but `productionServer.js` currently proxies `/api/*` directly and the checkout lacks the documented `/admin/login` route/auth client wiring. Treat the session-gated production proxy and admin-login route sections as planned session/auth contract language until `groupscout-site-0m0` restores or reconciles the implementation.

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
- Planned browser route rendering checks `/api/auth/status` before protected app routes and redirects unauthenticated users to `/admin/login` when admin auth is required.
- Planned `/admin/login` posts a setup token to `/api/auth/login`, verifies the resulting admin session, and then navigates to `/`.
- Planned protected app chrome includes a logout control that posts `/api/auth/logout`, clears local route state, and returns to `/admin/login`.
- The backend setup token is file-backed by default at `data/admin-setup-token`. It rotates after every successful login, so a second setup login needs the new file value.
- If the backend uses `ADMIN_SETUP_TOKEN`, that env-backed setup token is active and does not rotate automatically.
- The backend response includes a bearer `session_token` for non-browser smoke clients, but browser code relies on the HttpOnly `groupscout_session` cookie.
- Admin sessions are in-memory and expire after `ADMIN_SESSION_TTL_HOURS`, default `24`; backend restarts invalidate active browser sessions.
- The planned production request handler applies this authorization check before proxying `/api/*` to `UI_API_PROXY_TARGET`; with no session secret configured, the backend Docker smoke path can proxy `/api/*` without a browser login flow.
- Planned health, static, JSON, and proxied responses include baseline browser security headers: CSP, `x-content-type-options`, `x-frame-options`, and `referrer-policy`.
- Disabled UI access returns `404` so the operator UI is not mounted.
- `API_TOKEN` remains reserved for automation clients such as n8n and is not injected into browser requests.

For planned setup-token login, token rotation, logout, stale asset recovery, and direct curl smoke commands, use [Admin Token Flow](./admin-token-flow.md) as the canonical runbook. In the inspected current checkout, missing `/admin/login` is expected and is not stale-asset evidence.

## Design Tokens

Phase 9 does not add a new visual surface and does not require new `DESIGN.md` token names.

## TDD Evidence

- Red run: `node --test test/session-deployment.test.js test/api-boundary.test.js test/app-shell.test.js` failed because `web/src/server/uiDeployment.js` and `createMountedRouteShell(...)` did not exist.
- Targeted green run: `node --test test/session-deployment.test.js test/api-boundary.test.js test/app-shell.test.js`.
- Full-suite green run: `npm test`.
- Refresh red run on 2026-05-09: `node --test test/session-deployment.test.js` failed because the production request handler did not expose session-gated `/api/*` behavior.
- Refresh green run on 2026-05-09: `node --test test/session-deployment.test.js` passed after `createProductionRequestHandler(...)` gated `/api/*` with `authorizeUiApiRequest(...)` and added browser security headers.
- Admin completion green run on 2026-05-09: `node --test test/app-shell.test.js test/api-boundary.test.js test/phase-13-renderer-runtime.test.js test/session-deployment.test.js` and `npm test` passed after adding route auth checks, logout, login verification, and production readiness enforcement.
- Admin Docker smoke refresh on 2026-05-09: rebuilt backend/UI containers, verified `GET /api/auth/status` through the UI proxy returns `auth_required:true`, and bumped the immutable static asset query. The current asset/cache recovery value is maintained in [Admin Token Flow](./admin-token-flow.md).

## Out Of Scope

- Complex role matrices.
- Password, MFA, OAuth, or identity-provider UI.
- Durable multi-admin user management.
- Replacing automation `API_TOKEN` clients.
- Production reverse-proxy configuration.
- A server framework or browser renderer.
