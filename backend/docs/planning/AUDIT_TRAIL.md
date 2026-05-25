# Phase 27 — Input Audit & Verification Trail

**Goal:** Implement a system to store and track all raw inputs (PDFs, API responses, etc.) to allow for verification of the lead enrichment and scoring process. This builds trust by providing a direct link between an LLM-generated lead and the original source material.

## Brainstormed Tasks

### Part A — Storage Architecture
- [x] Define the `RawInput` schema for storing data (API response body, PDF content, source URL).
    - [x] Columns: `id` (UUID), `hash` (SHA256 of payload), `payload_type` (e.g. 'pdf', 'json', 'html'), `payload` (bytea/blob), `source_url` (text), `collector_name` (text), `created_at` (timestamp).
- [x] Choose storage backend: SQLite/Postgres for metadata + Disk/S3-compatible for larger payloads (PDFs).
    - [x] Start with database storage for simplicity, then move to disk if blob size becomes an issue.
- [x] Create database migration `006_audit_trail.up.sql` to add `raw_inputs` table.
- [x] Implement `internal/storage/audit.go` for managing raw input records.
    - [x] `Store(ctx, raw RawInput) (uuid.UUID, error)`
    - [x] `GetByID(ctx, id uuid.UUID) (*RawInput, error)`
    - [x] `GetByHash(ctx, hash string) (*RawInput, error)` — for de-duplication.

### Part B — Collector Integration
- [x] Update `Collector` interface to return raw data alongside `RawProject`.
    - [x] `RawProject` should have a `RawData []byte` field and a `RawType string` field.
- [x] Richmond: Store the raw PDF content and source URL for each run.
- [x] Delta: Store the raw PDF content and source URL.
- [x] BC Bid: Store the raw RSS XML and individual item descriptions.
- [x] News/Creative BC/VCC: Store the raw API/RSS responses.

### Part C — Enrichment Linking ✅ COMPLETE
- [x] Link `raw_input_id` to `leads` table to provide a direct link to the source data.
    - [x] Update migration `006_audit_trail.up.sql` to add `raw_input_id` to `leads` table.
    - [x] SQLite schema in `db.go` includes `raw_input_id TEXT REFERENCES raw_inputs(id)`.
- [x] Update `enricher.go` to store raw input before calling LLM.
    - [x] Check if hash exists first to avoid redundant storage (`auditStore.ExistsByHash`).
    - [x] Associate the `raw_input_id` with the generated `Lead` object (both enriched and skipped leads).
    - [x] All five fields captured: `Hash`, `PayloadType`, `Payload`, `SourceURL`, `CollectorName`.
- [x] Ensure `ExistsByHash` now points to the original raw input (`auditStore`, not `rawStore`).
- [x] TDD coverage (unit — `enricher_test.go`):
    - `TestToLeadRecord_rawInputIDPropagated` — `RawInputID` propagates through `toLeadRecord`.
    - `TestEnricher_processProject_storesAllAuditFields` — all five audit fields reach `AuditStore.Store`.
    - `TestEnricher_processProject_callsStoreAndLinksID` — enriched lead `RawInputID` matches store return.
    - `TestEnricher_processProject_linksIDToSkippedLead` — skipped leads also linked.
    - `TestEnricher_processProject_dedupCheck` — dedup uses audit store, not raw project store.
- [x] TDD coverage (integration — `leads_integration_test.go`):
    - `TestLeadStore_WithRawInputID` — lead + raw input FK round-trips in Postgres.

### Part D — Verification & Access ✅ COMPLETE
- [x] Create `GET /leads/{id}/raw` endpoint to retrieve the raw input used for a specific lead.
    - [x] Should return the raw payload with the correct `Content-Type`.
- [x] Add a CLI command `groupscout audit <lead_id>` to dump the raw input for manual verification.
    - [x] Flag `--save <path>` to save payload to a file.
    - [x] Flag `--meta` to show source URL, collector, and fetch time.
- [x] Update Slack notifications to include a link to the raw data (if exposed via internal reference).

### Part E — Retention & Privacy ✅ COMPLETE
- [x] Implement a scheduled cleanup worker that invokes retention policy for old raw inputs. Beads: `groupscout-site-j73`.
- [x] Add `PII_STRIP` option to remove sensitive info before storage if required.
- [x] Implement hashing logic to ensure we don't store identical payloads multiple times.

Raw audit storage, redaction policy, and opt-in automated cleanup scheduling are documented.

## Operations & Maintenance

### PII Redaction
By enabling `PII_STRIP=true`, the `AuditStore` will automatically redact emails and phone numbers from raw payloads before storage.
-   **Regex matching**: Uses predefined patterns to identify and replace sensitive strings.
-   **No impact on leads**: Redaction only affects the `raw_inputs` table, not the leads extracted from them (which are typically public anyway).

### Raw Audit Display Policy

Browser-safe sanitized preview implementation is tracked in `groupscout-site-4cv`.
Storage redaction and UI preview redaction are separate controls.

Current source caveat, 2026-05-25: the legacy `GET /leads/{id}/raw` endpoint returns the stored raw payload bytes and the inspected backend source snapshot does not enforce bearer or admin-session auth on that route. Do not expose it directly to browser UI. Use it only for deliberate operator verification until the authenticated UI alias and sanitized preview work land.

- Storage-time `PII_STRIP=true` protects persisted raw payloads by redacting emails and phone numbers before they are written to `raw_inputs`.
- Planned authenticated raw access through `GET /api/leads/{id}/raw` and CLI access through `groupscout audit <lead_id>` should return the stored bytes for deliberate operator verification. They are not the contract for inline browser preview. In the inspected backend source snapshot, legacy `GET /leads/{id}/raw` is still unauthenticated and should stay off the browser path.
- Inline browser preview must use a sanitized preview layer that redacts secrets, authorization headers, cookies, API keys, bearer/basic tokens, OAuth tokens, session IDs, signed URLs, webhook URLs, passwords, database URLs, provider keys, emails, phone/fax numbers, individual contact names, and residential/private-address details.
- Inline preview may preserve source metadata, collector name, collected timestamp, public permit or bid IDs, municipality, commercial project title, project value, public organization names, payload type, and hash metadata.
- JSON, XML/RSS, plain text, and sanitized text extracted from HTML are eligible for inline preview after redaction. PDFs, images, archives, office documents, unknown binary payloads, and oversized payloads are download/open-only.
- Tests for any preview implementation must fail on unredacted secrets or contact data, unsupported payload types rendered inline, raw body fields in default lead/detail responses, and browser-visible automation tokens.

### Data Retention (Purging)
Use `AUDIT_RETENTION_ENABLED=true` to start the server-mode retention worker, or run `groupscout audit-retention purge --days 30` for a one-time purge.
-   **Safety Guarantee**: Records referenced by a lead in the `leads.raw_input_id` column are NEVER purged.
-   **Recommended usage**: Enable the worker in Docker with a 30- or 60-day retention window and keep `AUDIT_RETENTION_RUN_ON_START=true` so missed downtime windows are caught on boot.

---

## Implementation Strategy — Agent Choreography

For a multi-agent execution of this phase, refer to [AGENT_CHOREOGRAPHY_PHASE27.md](./AGENT_CHOREOGRAPHY_PHASE27.md).

## Strict TDD Implementation Path
1.  **Test Storage Layer:** Write `audit_test.go` before `audit.go`. Assert `Store` and `GetByID` work.
2.  **Test Collector Changes:** Mock the collector to ensure it returns raw data.
3.  **Test Enrichment Integration:** Write a test where an enriched lead must have a valid `raw_input_id` pointing to the stored raw data.
4.  **Test API/CLI:** Assert `GET /leads/{id}/raw` returns the exact bytes stored.
