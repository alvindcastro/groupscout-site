# Smell Phase H0 - API Client Characterization Baseline

## Goal

Create a stable characterization baseline before the H1 API-client split. H0 does not refactor production code; it tightens tests so H1 fails on behavior changes rather than file layout changes.

Follow-up: H1 consumed this baseline on 2026-05-09 and completed the API-client split documented in [smell-h1-api-client-split.md](./smell-h1-api-client-split.md).

## Next Smell Phase

H1 targets the growing API client module:

- `web/src/api/client.js` currently owns same-origin transport rules, route builders, payload normalization, response adaptation, and feature-specific client methods.
- The next refactor should split feature adapters while preserving `createApiClient(...)` as the stable browser-facing entry point.
- The same-origin `/api/*` transport guard remains security-sensitive and should stay centralized.

## Affected Files For H1

- `web/src/api/client.js`
- `test/api-boundary.test.js`
- `test/lead-inbox-client.test.js`
- `test/lead-status-mutation-client.test.js`
- `test/raw-audit-client.test.js`
- `test/outreach-client.test.js`
- `test/pipeline-client.test.js`
- `test/stats-client.test.js`
- `test/alert-client.test.js`
- `test/system-client.test.js`
- `docs/developer-guide.md`
- `docs/testing.md`
- `docs/code-smell-transformation-prompts.md`
- `CHANGELOG.md`

## Behavior Map

- Public API facade and same-origin transport behavior: `test/api-boundary.test.js`
- Lead inbox route serialization, blank filter handling, sort metadata, and list response adaptation: `test/lead-inbox-client.test.js`
- Lead status PATCH routes, nullable payload intent, correction audit payloads, and correction validation: `test/lead-status-mutation-client.test.js`
- Raw audit alias route, encoded lead IDs, null policy defaults, and legacy endpoint rejection: `test/raw-audit-client.test.js`
- Outreach history/logging routes, optional draft payloads, required manual log fields, and no-send policy: `test/outreach-client.test.js`
- Pipeline history defaults, async run creation, default manual run reason, and optional actor payloads: `test/pipeline-client.test.js`
- Stats filters, summaries, source-yield hit-rate metadata, lead aging, verification quality, and required response sections: `test/stats-client.test.js`
- Alert filters, cursor route output, read-only metadata, required evidence/inventory/history sections, and absent mutation methods: `test/alert-client.test.js`
- System summary route, read-only metadata, required generated/pipeline/count sections, and absent mutation methods: `test/system-client.test.js`

## Public Contracts Named Before H1

- Named exports from `web/src/api/client.js`: `API_BASE_PATH`, `DEFAULT_LEAD_INBOX_SORT`, `LEAD_INBOX_ITEM_FIELDS`, and `createApiClient`.
- `createApiClient(...)` methods: `request`, `listLeads`, `getLead`, `patchLead`, `getLeadRawAudit`, `listLeadOutreach`, `logLeadOutreach`, `listPipelineRuns`, `startPipelineRun`, `getStats`, `listAlerts`, and `getSystem`.
- Browser request paths must be same-origin relative `/api/*` routes and must use `credentials: "same-origin"`.
- Browser requests must not inject automation-token headers.
- Read-only surfaces remain read-only: alert and system clients do not expose mutation helpers.

## Characterization Added

- Public module export and client method surface checks.
- Transport assertions for JSON defaults, caller header preservation, same-origin credential enforcement, non-JSON success handling, non-2xx errors, and pre-fetch rejection for invalid paths.
- Lead-scoped encoded route assertions for PATCH, raw audit, and outreach calls.
- Payload-shape assertions for blank lead filters, explicit sort overrides, nullable lead owner values, correction validation, outreach draft logging, and default pipeline run reasons.
- Adapter-shape assertions for raw audit policy defaults, pipeline health defaults, full stats policy metadata, alert required sections, and system required sections.

## Commands And Results

Focused H0 API-client baseline:

```sh
node --test test/api-boundary.test.js test/lead-inbox-client.test.js test/lead-status-mutation-client.test.js test/raw-audit-client.test.js test/outreach-client.test.js test/pipeline-client.test.js test/stats-client.test.js test/alert-client.test.js test/system-client.test.js
```

Result on 2026-05-09: passed 9 test files.

Full UI suite:

```sh
npm test
```

Result on 2026-05-09: passed all 22 test files.

## Red/Green Note

The H0 additions are characterization assertions against existing behavior, so they did not require production code changes. The targeted run stayed green because the current API client already implements the named behavior; future H1 changes should use these tests as the regression baseline.

H1 completion note on 2026-05-09: the same focused API-client command passed after the split, and `npm test` passed all 22 test files.
