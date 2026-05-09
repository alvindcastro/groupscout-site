# Phase 1 Lead Inbox API Contract And Client

Phase 1 defines the browser-safe lead inbox data boundary for the future Leads screen. It intentionally does not build the visual inbox table.

## Scope

- `GET /api/leads` is consumed through `createApiClient().listLeads(...)`.
- Browser code keeps using same-origin `/api/*` paths only.
- Components should consume the typed lead inbox response instead of hard-coding raw endpoint URLs.
- The contract represents the default priority sort for urgent, unowned, high-score leads.

## Query Contract

The client serializes:

- `q`
- `status`
- `source`
- `min_score`
- `created_from`
- `created_to`
- `property`
- `owner`
- `verification_state`
- `limit`
- `cursor`
- `sort=priority` by default

## Lead List Item Contract

The response adapter exposes:

- `id`
- `score`
- `title`
- `segment`
- `projectType`
- `location`
- `propertyFit`
- `source`
- `estimatedCrewSize`
- `estimatedDurationDays`
- `outreachTiming`
- `status`
- `owner`
- `createdAt`
- `evidenceState`
- `verificationState`

## TDD Evidence

- Red run: `npm test` failed because the Phase 1 lead inbox contract exports did not exist yet.
- Green run: `npm test` passes after adding the smallest client and adapter surface. After H1, callers still enter through `web/src/api/client.js`, and the lead adapter implementation lives in `web/src/api/leads.js`.
