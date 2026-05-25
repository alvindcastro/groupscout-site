# Phase 0 Product Contract And Test Harness

## Goal

Establish the UI implementation contract and testing harness needed for the lead-management MVP without building product features.

## User Workflow Reserved

The first durable workspace will support operators moving through lead triage, review, ownership, evidence, outreach history, and outcomes. Phase 0 only reserves the shell areas required to host that workflow later:

- Today
- Leads
- Verification
- Outreach
- Pipeline
- Analytics
- Settings

## Acceptance Criteria Status

- Stable design-token location: covered by `test/design-tokens.test.js`.
- Browser API boundary: covered by `test/api-boundary.test.js`.
- Planned IA shell: covered by `test/app-shell.test.js`.
- Phase 0 had no feature workflow implementation: the shell returned placeholder content only until later phase screen models replaced the route placeholders.

## Red-Green Notes

Initial targeted command:

```sh
npm test
```

Expected red result:

- `web/src/design/tokens.js` was missing.
- `web/src/app/shell.js` was missing.
- `web/src/api/client.js` was missing.

Final green result:

- `npm test` passes all Phase 0 guardrail tests.

## Out Of Scope

- Lead inbox rendering
- Lead detail rendering
- Real API calls to product endpoints
- Analytics
- Auth/session wrapper
- Deployment behavior
