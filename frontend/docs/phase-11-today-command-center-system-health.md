# Phase 11 Today Command Center And System Health Summary

Phase 11 turns the reserved `Today` route into the operator command center. It summarizes the day’s highest-priority work while keeping every action routed to the owning workspace.

## Scope

- `web/src/app/todayCommandCenter.js` owns the Today screen model, mocked command-center data, summary counts, priority lead rows, aging claimed rows, active alert snippets, failed job rows, system health metadata, responsive metadata, and read-only action policy.
- `createRouteShell("/")` mounts Today instead of the Phase 0 placeholder.
- `createApiClient().getSystem()` reads a UI-friendly system summary through same-origin `GET /api/system`.
- `createSystemSummaryRequest()` documents the read-only browser request intent.

## Command Center Sections

- Summary: high-score new leads, aging claimed leads, active alerts, failed jobs, and overall system health.
- Priority leads: lead title, score, owner, status, source, timing, and link to the Lead Detail workspace.
- Aging claimed work: owner, age, status, next step, and link back to the lead.
- Active alerts: property, SPS, state, impact, and link to the read-only Alertd console.
- Failed jobs: collector, status, failure time, reason, and link to Pipeline.
- System health: API, database, collector freshness, and LLM enrichment.

## Read-Only Policy

- Today is an orientation and routing surface.
- Claim, dismiss, outreach, alert, and pipeline actions stay in their owning workspaces.
- The browser client exposes `getSystem()` only for system summary reads; it does not expose system settings writes, restarts, or admin mutations.

## Design Tokens

Phase 11 reuses existing documented `DESIGN.md` component tokens:

- `card-base` for compact summary and health surfaces.
- `feature-comparison-table` for dense work lists.
- `property-row` for rows.
- `badge-tag` and `badge-type` for compact labels.
- `search-pill` and `button-secondary` for scope controls and navigation actions.

No new `DESIGN.md` token names were required.

## TDD Evidence

- Red run: `node --test test/today-command-center.test.js test/system-client.test.js test/app-shell.test.js` failed because `web/src/app/todayCommandCenter.js`, `/` shell mounting, and `createApiClient().getSystem()` did not exist.
- Targeted green run: `node --test test/today-command-center.test.js test/system-client.test.js test/app-shell.test.js`.
- Full-suite green run: `npm test`.

## Out Of Scope

- New status, outreach, alert, pipeline, or settings mutations.
- A custom dashboard builder.
- Replacing the deeper Leads, Verification, Outreach, Pipeline, Analytics, or Alerts workspaces.
- Direct browser calls to automation endpoints, health endpoints outside `/api/*`, worker internals, or backend storage.
