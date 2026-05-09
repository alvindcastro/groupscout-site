# Phase 10 Later Alertd Read-Only Console

Phase 10 adds a later-stage, read-only disruption alert console. It keeps `alertd` Slack-first for interrupt delivery while giving operators a durable place to inspect current alert state, evidence, room inventory, and action history.

## Scope

- `web/src/app/alertdConsole.js` owns the Alertd console screen model, mocked alert data, SPS summaries, evidence rows, room inventory details, action-history rows, responsive metadata, and read-only action policy.
- `createRouteShell("/alerts")` mounts the Alerts screen instead of a placeholder.
- `createApiClient().listAlerts(...)` reads alert data through same-origin `GET /api/alerts`.
- `createAlertConsoleRequest(...)` documents the read-only browser request intent for state, property, limit, and cursor filters.

## Alert Fields

- Summary: active alert count, highest SPS, active unavailable-room impact, and last update.
- Alert rows: property, SPS, state, impact, and updated timestamp.
- Evidence: evidence type, label, value, source URL, and observed timestamp.
- Room inventory: total, unavailable, available, out-of-service, and last-updated values when supplied by the contract.
- Action history: actor, action, channel, note, and created timestamp.

## Read-Only Policy

- Slack remains the interrupt channel for the lead-management MVP.
- The console is a monitoring surface only.
- Acknowledge, resolve, and suppress actions are represented as disabled out-of-scope actions.
- The browser client exposes `listAlerts(...)` only; it does not expose alert create, patch, acknowledge, resolve, or suppress methods.

## Design Tokens

Phase 10 reuses existing documented `DESIGN.md` component tokens:

- `card-base` for compact alert and summary surfaces.
- `feature-comparison-table` for alert, evidence, and action-history tables.
- `property-row` for dense rows.
- `badge-tag` and `badge-type` for compact labels.
- `search-pill` and `button-secondary` for filters and refresh controls.

No new `DESIGN.md` token names were required.

## TDD Evidence

- Lead-workflow stability check: `npm test` passed before Phase 10 implementation.
- Red run: `node --test test/alert-console.test.js test/alert-client.test.js test/app-shell.test.js test/api-boundary.test.js` failed because `web/src/app/alertdConsole.js`, `/alerts` shell mounting, and `createApiClient().listAlerts(...)` did not exist.
- Targeted green run: `node --test test/alert-console.test.js test/alert-client.test.js test/app-shell.test.js test/api-boundary.test.js`.
- Full-suite green run: `npm test`.

## Out Of Scope

- Replacing Slack as the alert interrupt channel.
- Alert mutations, acknowledgements, suppressions, or resolutions.
- Direct browser calls to alertd internals or automation-only endpoints.
- Inline backend worker controls.
- A full alertd administration console.
