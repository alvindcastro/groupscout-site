# Phase 5 Verification Queue And Raw Audit Review

Phase 5 adds a focused Verification Queue for high-value or low-trust leads and moves raw audit review links to the UI-safe `GET /api/leads/{id}/raw` alias.

## Scope

- `web/src/app/verificationQueue.js` owns queue trigger classification, mocked queue data, filters, row action intent, responsive metadata, and raw audit review metadata.
- `createRouteShell("/verification")` mounts the Verification Queue instead of a placeholder.
- `createApiClient().getLeadRawAudit(...)` reads raw evidence through same-origin `GET /api/leads/{id}/raw`.
- Lead Detail source evidence now points at `/api/leads/{id}/raw` so detail and verification surfaces use the same UI-safe raw audit boundary.

## Queue Trigger Model

Verification triggers are explicit and test-covered:

- Missing source URL or raw audit record.
- High score with weak AI rationale or low AI confidence.
- Contradiction between raw source fields and enriched fields.
- Low confidence collector parse.
- Manual operator flag.

Rows are included only when at least one trigger is present.

## UI Behavior

- Screen model: `createVerificationQueueScreen(...)`.
- Controls: trigger, source, owner, minimum score, and clear filters.
- Table columns: score, lead, trigger, source, owner, raw audit, and updated.
- Row actions: verify, correct, dismiss, and return to lead.
- Responsive behavior: desktop verification table, tablet table with lower-priority columns hidden, and mobile verification cards. The renderer uses the mobile card model directly, so mobile output keeps the queue count, lead title, trigger, raw audit link, and row actions visible even though desktop table rows are intentionally empty in mobile mode.
- Loading, empty, and error states are represented in the screen model.

## Raw Audit Access

- Raw audit links use `/api/leads/{id}/raw`.
- The screen declares the endpoint as `GET /api/leads/{id}/raw`.
- Raw payloads are not rendered inline in Phase 5.
- Direct or legacy raw endpoints remain outside browser-facing screen links.

## Redaction Policy

Redaction rules are not defined yet. Phase 5 keeps that decision visible through `RAW_AUDIT_REDACTION_POLICY`:

- `status`: `blocked`
- `todo`: `Define raw audit payload redaction rules before rendering sensitive fields inline.`

The UI can open the authenticated raw audit entry point, but it does not claim complete safe inline raw payload display.

## Design Tokens

The queue uses existing documented `DESIGN.md` component tokens:

- `search-pill` for filter surfaces.
- `text-input` for input/select controls.
- `feature-comparison-table` and `property-row` for dense operational table structure.
- `badge-tag` and `badge-type` for trigger/type metadata.
- `button-primary` for the verify action.
- `button-secondary` for secondary review actions.

## TDD Evidence

- Red run: `node --test test/verification-queue.test.js test/raw-audit-client.test.js test/app-shell.test.js test/lead-detail-screen.test.js` failed on missing `web/src/app/verificationQueue.js`, missing `createApiClient().getLeadRawAudit(...)`, placeholder `/verification` routing, and the legacy Lead Detail raw audit path.
- Targeted green run: `node --test test/verification-queue.test.js test/raw-audit-client.test.js test/app-shell.test.js test/lead-detail-screen.test.js`.
- Full-suite green run: `npm test`.
- Renderer review refresh on 2026-05-09: `node --test test/verification-queue.test.js test/phase-13-renderer-runtime.test.js` covered the mobile Verification Queue card model and rendered mobile route output.

## Out Of Scope

- Inline raw payload rendering before redaction rules are defined.
- Backend redaction implementation.
- Bulk verification actions.
- Outreach logging or automated outreach.
- Complex permissions for who can verify, correct, or dismiss.
