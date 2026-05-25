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

Raw audit links are allowed before inline previews because they open the authenticated raw audit route deliberately. Inline preview is a stricter feature and must not ship until the preview renderer applies this policy.

`RAW_AUDIT_REDACTION_POLICY` is now defined as:

- `status`: `defined`
- `inline_preview_default`: `disabled`
- `raw_access_endpoint`: `GET /api/leads/{id}/raw`
- `preview_payload_types`: `application/json`, `application/xml`, `text/xml`, `application/rss+xml`, `text/plain`, and sanitized text extracted from `text/html`
- `download_only_payload_types`: `application/pdf`, images, archives, office documents, unknown binary payloads, and any payload larger than the configured preview limit

Inline preview redaction must remove or mask these values before rendering:

- Secrets and credentials: `Authorization`, `Cookie`, `Set-Cookie`, API keys, bearer/basic tokens, OAuth tokens, session IDs, signed URLs, webhook URLs, passwords, database URLs, and provider keys.
- Direct contact fields: email addresses, phone and fax numbers, personal contact names, and contact-specific fields such as `contact_email`, `contact_phone`, `applicant_email`, `contractor_phone`, `owner_phone`, `first_name`, and `last_name`.
- Residential or private-address details when a payload appears to describe an individual residence rather than a public commercial project.
- Oversized free-text source excerpts beyond the preview limit; render a truncation marker instead of the hidden text.

Inline preview may preserve business-safe evidence fields needed for verification: source name, source URL with sensitive query parameters stripped, collector name, collected timestamp, public permit or bid identifiers, municipality, commercial project title, project value, public organization names, payload type, and hash metadata.

Authenticated raw audit access differs from inline evidence display:

- `GET /api/leads/{id}/raw` returns the stored raw bytes with the stored content type through the API/session boundary.
- Lead Detail and Verification Queue screens may link to that route, but they must not fetch it during default render, search indexing, mocked fixture loading, or screen-model construction.
- A future inline preview must use a separate sanitized preview adapter or response envelope. It must not reuse raw bytes directly in browser-visible state. Tracked follow-up: `groupscout-site-4cv`.

Tests that must block unsafe payload exposure:

- Fixture and screen-model tests fail if default detail or queue data contains raw body fields such as `payload`, `raw_body`, `raw_data`, `html`, `pdf`, `body`, or `bytes`.
- Raw audit client tests cover same-origin `/api/leads/{id}/raw`, encoded lead IDs, and explicit user action before fetch.
- Preview tests cover redaction of secrets, bearer tokens, cookies, signed URLs, API keys, database URLs, webhook URLs, emails, phones, and individual contact names.
- Preview tests cover payload-type gating: JSON/XML/plain text/sanitized HTML can render after redaction; PDF, image, archive, office, unknown binary, and oversized payloads render metadata plus a download/open action only.
- Static asset scans continue to fail on browser-visible `API_TOKEN`, provider keys, database URLs, Slack/email secrets, session secrets, or literal bearer tokens.

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
