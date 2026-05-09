# Phase 6 Outreach Workspace And Activity Log

Phase 6 adds manual outreach drafting, contact capture, outreach attempt logging, and outcome history without sending email or syncing a CRM from the UI.

## Scope

- `web/src/app/outreachWorkspace.js` owns the standalone Outreach Workspace screen model, mocked lead outreach data, draft/contact fields, manual display states, outcome options, responsive metadata, and activity timeline rows.
- `createRouteShell("/outreach")` mounts the Outreach Workspace instead of a placeholder.
- `web/src/app/leadDetail.js` embeds a compact outreach workspace inside Lead Detail so operators can draft, copy, mark manually sent, log attempts, and see outreach outcomes next to evidence.
- `createApiClient().listLeadOutreach(...)` reads outreach history through same-origin `GET /api/leads/{id}/outreach`.
- `createApiClient().logLeadOutreach(...)` logs manual attempts through same-origin `POST /api/leads/{id}/outreach`.

## Manual Outreach Behavior

- Draft fields stay editable and include channel, contact, subject/message or draft text, notes, and outcome metadata depending on the surface.
- Copy, marked-sent, and logged states are display states only. They do not send email from GroupScout.
- Logging requires channel, contact details, notes, and one of the Phase 6 outcomes.
- Supported outcomes are `contacted`, `won`, `lost`, and `no_response`.

## Activity History

- Lead Detail activity now includes outreach attempt and outreach outcome entries.
- The standalone Outreach Workspace renders outreach attempts as timeline rows with channel, contact, notes, outcome label, formatted timestamp, and `manualOnly: true`.
- Outcome capture is designed to feed later analytics, but Phase 6 does not implement analytics.

## Design Tokens

Phase 6 uses existing documented `DESIGN.md` component tokens:

- `card-base` for workspace sections.
- `text-input` for editable contact, draft, notes, and outcome controls.
- `button-primary` for manual log actions.
- `button-secondary` for copy and mark-sent actions.
- `badge-tag` and `badge-type` for outcome and type metadata.
- `property-row` for timeline/activity rows.

## TDD Evidence

- Red run: `npm test` failed on missing outreach client methods and missing Lead Detail outreach workspace behavior.
- Targeted red run: `node test/app-shell.test.js` failed while `/outreach` still returned a placeholder.
- Full-suite green run: `npm test`.

## Out Of Scope

- Automated email sending.
- CRM sync.
- Bulk outreach.
- Browser rendering or clipboard integration.
- Analytics based on outreach outcomes.
