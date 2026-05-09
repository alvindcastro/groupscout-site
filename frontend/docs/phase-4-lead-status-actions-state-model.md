# Phase 4 Lead Status Actions And State Model

Phase 4 adds the recommended v1 lead status model, valid action metadata, Lead Detail action controls, and the typed PATCH boundary needed to mutate lead state without spreading endpoint details through UI code.

## Scope

- `web/src/app/leadStatus.js` owns the v1 workflow statuses and action transition table.
- `createLeadDetailScreen(...)` renders only actions allowed for the current lead status.
- `createLeadDetailScreen(...).actions.submit(...)` validates actions before calling the mutation boundary.
- `createApiClient().patchLead(...)` serializes `PATCH /api/leads/{id}` payloads for status, owner, notes, snooze date, correction reason, and safe field corrections.
- `follow_up` and `corrected` remain actions, not statuses. They preserve the current status unless the backend contract later adds dedicated states.

## State Model

Recommended v1 statuses:

- `new`
- `notified`
- `claimed`
- `contacted`
- `snoozed`
- `flagged`
- `verified`
- `dismissed`
- `won`
- `lost`
- `no_response`

Allowed actions:

- `new`, `notified`: `claim`, `dismiss`, `snooze`, `flag`
- `claimed`: `contacted`, `won`, `lost`, `no_response`, `snooze`
- `contacted`: `won`, `lost`, `no_response`, `follow_up`
- `snoozed`: `reopen`, `dismiss`
- `flagged`: `verified`, `corrected`, `dismiss`
- `verified`: `claim`, `dismiss`
- `dismissed`, `won`, `lost`: `reopen`
- `no_response`: `follow_up`, `lost`, `snooze`

## Mutation Behavior

- Invalid transitions are rejected before `patchLead(...)` is called.
- `claim` requires an owner.
- `dismiss`, `flag`, `contacted`, `won`, `lost`, `no_response`, `follow_up`, `reopen`, and `verified` require notes.
- `snooze` requires a `YYYY-MM-DD` snooze date.
- `corrected` requires at least one correction plus a correction reason.
- Corrections preserve original AI and source values with actor and reason metadata, and the API client serializes them under `corrections` instead of overwriting source-backed fields directly.

## Design Tokens

Action controls continue to use documented `DESIGN.md` component tokens:

- `button-primary` for primary positive workflow actions.
- `button-secondary` for secondary, review, or outcome actions.
- Existing detail workspace tokens remain unchanged for evidence and correction display.

Mint green remains reserved for active or confirmation accents. Phase 4 does not add bulk actions.

## TDD Evidence

- Red runs failed first for missing state helpers, missing PATCH client support, read-only Phase 3 actions, and missing correction reason serialization.
- Targeted green run: `node --test test/lead-detail-screen.test.js test/lead-status-state-model.test.js test/lead-status-mutation-client.test.js`.
- Full-suite green run: `npm test`.
