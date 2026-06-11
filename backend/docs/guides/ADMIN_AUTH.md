# Admin Auth And Setup Token Flow Contract

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/backend/docs/guides/ADMIN_AUTH.md`. Run backend commands from `/mnt/c/Users/alvin/GolandProjects/groupscout`; run UI commands from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

Status reconciliation, 2026-05-25: this is a contract-only planned admin auth/session runbook. The inspected backend source snapshot does not currently expose `ADMIN_AUTH_*` config, setup-token file creation, `groupscout_session`, or `/api/auth/*` routes, and the inspected UI checkout does not currently expose `/admin/login`. Backend implementation is tracked by `groupscout-site-eqm`; UI restoration/reconciliation is tracked by `groupscout-site-0m0`.

## Planned Runtime Defaults

Admin auth should be enabled by default when this contract is implemented:

```env
ADMIN_AUTH_ENABLED=true
ADMIN_SETUP_TOKEN_FILE=data/admin-setup-token
ADMIN_SESSION_TTL_HOURS=24
```

If `ADMIN_SETUP_TOKEN` is not set, the backend reads or creates `ADMIN_SETUP_TOKEN_FILE`. The generated file is written with restrictive permissions and contains the first setup token.

If `ADMIN_SETUP_TOKEN` is set, that value is used as the setup token instead of the file. Env-backed setup tokens cannot rotate automatically after login; rotate them by changing the env var and restarting the backend.

## Browser Login

1. Start or rebuild the backend and UI.
2. Read the current setup token.
3. Open the UI login route.
4. Submit the token.

Docker setup-token command:

```bash
docker exec groupscout_app sh -lc 'cat data/admin-setup-token'
```

Local backend setup-token command:

```bash
cat /mnt/c/Users/alvin/GolandProjects/groupscout/data/admin-setup-token
```

Login URL:

```txt
http://localhost:3001/admin/login
```

After successful login, the planned backend sets an HttpOnly `groupscout_session` cookie. Browser JavaScript cannot read or clear this cookie directly; logout must call `POST /api/auth/logout`.

## Token Rotation

File-backed setup tokens rotate after successful login:

- The submitted setup token becomes invalid.
- `data/admin-setup-token` is replaced with a new token.
- A second login attempt needs the new file value.

Read the token again after a successful login:

```bash
docker exec groupscout_app sh -lc 'cat data/admin-setup-token'
```

Env-backed setup tokens do not rotate automatically:

```env
ADMIN_SETUP_TOKEN=your-temporary-setup-token
```

Use env-backed setup tokens only when you have an external secret-rotation path.

## API Flow

Status:

```bash
curl -i http://localhost:8080/api/auth/status
```

Expected unauthenticated response when auth is enabled:

```json
{"auth_required":true,"authenticated":false,"setup_token_file":"data/admin-setup-token"}
```

Login with a setup token and keep the session cookie:

```bash
curl -i -c /tmp/groupscout-admin.cookies \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_SETUP_TOKEN"}' \
  http://localhost:8080/api/auth/login
```

The login response includes `session_token`, `token_type:"Bearer"`, `expires_at`, `setup_token_rotated`, and `user`. Browser clients use the `groupscout_session` cookie. Non-browser smoke clients can use the returned `session_token` as a bearer token for admin-session endpoints:

```bash
curl -i -H "Authorization: Bearer YOUR_SESSION_TOKEN" http://localhost:8080/api/auth/me
```

Cookie-backed current admin:

```bash
curl -i -b /tmp/groupscout-admin.cookies http://localhost:8080/api/auth/me
```

Logout and revoke the current session:

```bash
curl -i -b /tmp/groupscout-admin.cookies -c /tmp/groupscout-admin.cookies \
  -X POST http://localhost:8080/api/auth/logout
```

## Docker UI Smoke

Rebuild when auth routes or static assets changed:

```bash
cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui
npm run build
docker compose -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml -f compose.dev.yml up -d --build groupscout-ui
```

Verify the UI proxy can reach backend auth:

```bash
curl -i --max-time 5 http://localhost:3001/api/auth/status
curl -i --max-time 5 http://localhost:3001/
```

Expected current HTML references `/assets/app.js?v=admin-login-1`. If a browser still shows the old app, hard-refresh or open `/admin/login` directly.

## Security Boundary

- `API_TOKEN` remains an automation token for n8n and server-to-server triggers.
- Do not put `API_TOKEN`, provider keys, database URLs, or `UI_SESSION_SECRET` in browser-visible code, static assets, public config, or UI Compose env.
- Admin browser access uses the `groupscout_session` cookie and backend session store.
- Current sessions are in memory. Restarting the backend invalidates active sessions.
