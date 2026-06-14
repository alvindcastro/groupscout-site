# Smell Phase H1 - API Client Split

## Goal

Reduce the growing browser API client module into focused API adapters while preserving the stable `createApiClient(...)` facade and the centralized same-origin `/api/*` transport guard.

## Module Ownership

- `web/src/api/client.js`: public facade and stable exports.
- `web/src/api/transport.js`: `API_BASE_PATH`, same-origin `/api/*` validation, JSON request defaults, session credentials, non-2xx errors, and non-JSON success handling.
- `web/src/api/shared.js`: small adapter helpers for query params, lead ID validation, own-property checks, and blank-value checks.
- `web/src/api/leads.js`: lead inbox reads, lead PATCH writes, lead payload normalization, and lead inbox response adaptation.
- `web/src/api/rawAudit.js`: UI-safe raw audit reads for `GET /api/leads/{id}/raw`.
- `web/src/api/outreach.js`: outreach history reads and manual outreach attempt logging.
- `web/src/api/pipeline.js`: pipeline run history reads and async manual run creation.
- `web/src/api/stats.js`: analytics stats reads and source-yield hit-rate policy metadata.
- `web/src/api/alerts.js`: read-only alert list adaptation.
- `web/src/api/system.js`: read-only system health summary adaptation.

## Preserved Public Contracts

- Callers still import from `web/src/api/client.js`.
- Named exports remain `API_BASE_PATH`, `DEFAULT_LEAD_INBOX_SORT`, `LEAD_INBOX_ITEM_FIELDS`, and `createApiClient`.
- `createApiClient(...)` still exposes `request`, `listLeads`, `getLead`, `patchLead`, `getLeadRawAudit`, `listLeadOutreach`, `logLeadOutreach`, `listPipelineRuns`, `startPipelineRun`, `getStats`, `listAlerts`, and `getSystem`.
- Browser requests still fail before fetch for absolute `http` or `https` URLs, empty paths, and paths outside `/api/`.
- Browser requests still use `credentials: "same-origin"` and do not inject automation-token headers.
- Alert and system surfaces remain read-only.

## Commands And Results

Focused H1 API-client split run:

```sh
node --test test/api-boundary.test.js test/lead-inbox-client.test.js test/lead-status-mutation-client.test.js test/raw-audit-client.test.js test/outreach-client.test.js test/pipeline-client.test.js test/stats-client.test.js test/alert-client.test.js test/system-client.test.js
```

Result on 2026-05-09: passed 9 test files.

Full UI suite:

```sh
npm test
```

Result on 2026-05-09: passed all 22 test files.

## Next Smell Phase

H2 should isolate mock fixtures from runtime screen factories. The H1 split does not introduce live backend calls, rendering, or fixture movement.
