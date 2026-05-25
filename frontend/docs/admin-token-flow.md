# Admin Token Flow

This document is maintained in the coordinator repo at `/mnt/c/Users/alvin/groupscout-site/frontend/docs/admin-token-flow.md`. Run UI commands from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`; run backend commands from `/mnt/c/Users/alvin/GolandProjects/groupscout`.

Status reconciliation, 2026-05-25: this is the planned admin-token runbook. The inspected backend source snapshot does not expose `/api/auth/*`, and the inspected UI checkout does not expose `/admin/login` or the documented auth client wiring. Backend implementation is tracked by `groupscout-site-eqm`; UI restoration/reconciliation is tracked by `groupscout-site-0m0`.

## Operator Login Steps

Use this flow when testing the Docker UI at `http://localhost:3001`.

1. Confirm auth is enabled:

   ```sh
   curl -i --max-time 5 http://localhost:3001/api/auth/status
   ```

   Expected unauthenticated response:

   ```json
   {"auth_required":true,"authenticated":false,"setup_token_file":"data/admin-setup-token"}
   ```

2. Read the setup token from the backend container.

   The in-container token file is `/app/data/admin-setup-token`. From the host, use:

   ```sh
   docker exec groupscout_app sh -lc 'cat data/admin-setup-token'
   ```

3. Open:

   ```txt
   http://localhost:3001/admin/login
   ```

   The setup-token form should appear as a compact floating window centered in the workspace area. If it renders as a wide page panel, rebuild the UI static assets and Docker service before continuing the browser test.

4. Submit the token. Successful login sets the backend-owned HttpOnly `groupscout_session` cookie and navigates to `/`.

5. Use the topbar `Log out` control to call `/api/auth/logout`, revoke the session, expire the cookie, and return to `/admin/login`.

## Rotation Rules

The default setup token is file-backed by `/app/data/admin-setup-token` inside `groupscout_app`. The auth status response reports the same file as the relative backend path `data/admin-setup-token`. It rotates after every successful login.

- If a token is rejected after a successful login, read `/app/data/admin-setup-token` again with `docker exec groupscout_app sh -lc 'cat data/admin-setup-token'`.
- If `ADMIN_SETUP_TOKEN` is configured in the backend environment, the token is env-backed and cannot rotate automatically. Change the env var and restart the backend to rotate it.
- Sessions expire after `ADMIN_SESSION_TTL_HOURS`, default `24`.
- Sessions are currently in memory, so backend restarts invalidate active browser sessions.

## Docker Cache Notes

The UI serves immutable `/assets/*` files. The admin-login Docker refresh uses:

```txt
/assets/app.js?v=admin-login-2
```

If the browser does not show login after containers were rebuilt:

```sh
curl -i --max-time 5 http://localhost:3001/
```

The HTML should reference `admin-login-2`. If it still references an older version such as `pipeline-output-4`, rebuild the UI image. If it references `admin-login-2` but the browser is stale, hard-refresh or open `/admin/login` directly.

## Direct API Smoke

Login without a browser:

```sh
TOKEN="$(docker exec groupscout_app sh -lc 'cat data/admin-setup-token')"
curl -i -c /tmp/groupscout-admin.cookies \
  -H "Content-Type: application/json" \
  -d "{\"token\":\"${TOKEN}\"}" \
  http://localhost:3001/api/auth/login
curl -i -b /tmp/groupscout-admin.cookies http://localhost:3001/api/auth/me
```

Logout:

```sh
curl -i -b /tmp/groupscout-admin.cookies -c /tmp/groupscout-admin.cookies \
  -X POST http://localhost:3001/api/auth/logout
```

Do not use backend `API_TOKEN` in browser code. Browser auth is cookie/session based.
