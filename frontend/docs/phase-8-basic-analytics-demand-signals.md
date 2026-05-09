# Phase 8 Basic Analytics And Demand Signals

Phase 8 adds a basic operator analytics surface for explainable lead operations. It stays operational and fixed-scope: source quality, pipeline coverage, lead aging, verification quality, and upcoming demand signals.

## Scope

- `web/src/app/analyticsDashboard.js` owns the Analytics screen model, mocked stats data, source-yield calculations, demand rows, responsive metadata, and metric-definition labels.
- `createRouteShell("/analytics")` mounts the Analytics screen instead of a placeholder.
- `createApiClient().getStats(...)` reads analytics through same-origin `GET /api/stats`.
- `createStatsRequest(...)` documents the browser request intent for date range, segment, and property filters.

## Stats Fields

- Date range and denominator labels are visible on the screen.
- Summary groups cover status, source, score band, owner, and week.
- Source yield includes total leads, claimed rate, won rate, lost rate, hit rate, and source-specific denominator.
- Lead aging buckets show operational backlog age.
- Verification quality shows reviewed-lead counts and rates.
- Demand rows group upcoming lead volume and estimated room nights by week, segment, and property.

## Source Hit-Rate Definition

Phase 8 defines source hit rate as:

```text
won leads / total source leads
```

The numerator is only leads with `won` status. The denominator is all leads from that source collected in the selected date range. Claimed, contacted, lost, no-response, and dismissed leads are visible in related metrics but excluded from the hit-rate numerator.

## Design Tokens

Phase 8 reuses existing documented `DESIGN.md` component tokens:

- `card-base` for compact summary and metric surfaces.
- `feature-comparison-table` for source yield, verification quality, and demand tables.
- `property-row` for dense rows.
- `badge-tag` and `badge-type` for compact metric labels.
- `search-pill`, `text-input`, and `button-secondary` for filters and refresh controls.

No new `DESIGN.md` token names were required.

## TDD Evidence

- Red run: `node --test test/stats-client.test.js test/analytics-dashboard.test.js` failed because `web/src/app/analyticsDashboard.js` and `createApiClient().getStats(...)` did not exist.
- Targeted green run: `node --test test/stats-client.test.js test/analytics-dashboard.test.js test/analytics-screen.test.js test/app-shell.test.js test/api-boundary.test.js`.
- Full-suite green run: `npm test`.

## Out Of Scope

- Custom dashboard builder.
- Ad hoc chart configuration.
- CRM replacement analytics.
- Direct database access from the browser.
- Unexplained source-quality metrics without visible denominator and date-range context.
