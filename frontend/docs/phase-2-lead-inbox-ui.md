# Phase 2 Lead Inbox UI

Phase 2 builds the first operator-facing Lead Inbox screen using mocked lead data and the Phase 1 client field shape. The implementation keeps the browser surface behind app modules and does not add status mutations, outreach logging, raw audit viewing, or analytics.

## Scope

- `createLeadInboxScreen(...)` returns a dense Lead Inbox screen model from `web/src/app/leadInbox.js`.
- `/leads` mounts the Lead Inbox through `createRouteShell("/leads")`.
- Mocked leads use the adapted Phase 1 field names: score, title, segment, project type, property fit, owner, created date, evidence state, and verification state.
- Filtering is represented in query state for search, status, source, minimum score, created date range, property, owner, and verification state.
- Row selection returns detail navigation intent for `/leads/{id}` without implementing the detail workspace.

## UI Behavior

- Desktop mode exposes the full dense operations table.
- Tablet mode keeps priority columns visible and collapses lower-priority source and created-date fields.
- Mobile mode uses lead cards with preserved score, title, location/property, outreach timing, and evidence/verification metadata.
- Loading, empty, and error states are distinct and use status or alert roles in the screen metadata.
- Controls expose accessible labels, a stable focus order, and minimum touch-target metadata.

## Design Tokens

The screen references documented `DESIGN.md` component tokens:

- `text-input`
- `button-secondary`
- `search-pill`
- `segmented-tab-active`
- `badge-tag`
- `badge-type`
- `feature-comparison-table`
- `property-row`

Mint green remains reserved for active or confirmation accents and is not used for large surfaces or body text.

## TDD Evidence

- Red run: `node --test test/lead-inbox-screen.test.js` failed because `web/src/app/leadInbox.js` did not exist.
- Follow-up red run: `npm test` failed while `/leads` still returned placeholder content and Phase 2 component tokens were missing.
- Green run: `npm test` passes all Phase 0, Phase 1, and Phase 2 tests.

## Current Harness Limits

The repo still uses Node's built-in `node:test` runner without a DOM or browser dependency. Phase 2 therefore verifies render contracts, responsive metadata, accessibility metadata, and token references, but not computed CSS, real media queries, or browser focus traversal.
