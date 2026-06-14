# Code Smells And Housekeeping Notes

These are documentation-only findings from the current UI and backend housekeeping pass. They are not code changes.

For future implementation prompts that transform these smells under strict TDD, see [code-smell-transformation-prompts.md](./code-smell-transformation-prompts.md).

## UI Repo

### Growing API Client Module - Mitigated In H1

H1 split `web/src/api/client.js` into a stable facade plus focused API modules under `web/src/api/`. `web/src/api/transport.js` now owns the centralized same-origin `/api/*` guard, and feature adapters own their own paths, payload normalization, and response adaptation.

Remaining risk: each new browser API method should still land in the focused adapter that owns its feature area, otherwise the facade or shared helpers can start growing again.

Tracked follow-up: keep `createApiClient(...)` as the public entry point, keep transport behavior centralized, and add or update focused adapter tests when new API surfaces land. Use `groupscout-site-2h1` for smell-refactor hygiene and `groupscout-site-29q` for generated client/type work.

### Mock Fixture Coupling

Production screen modules export and default to mock data such as lead rows, detail evidence, analytics stats, alert rows, and Today summaries.

Impact: demo/test data is coupled to runtime factories, which can blur the line between fixture behavior and production behavior once live data is introduced.

Tracked follow-up: move fixtures behind explicit test/demo inputs before live backend wiring or renderer expansion ships under `groupscout-site-2h1`.

### Duplicated Domain Constants

Some domain rules exist in both app model modules and the API client layer. Examples include outreach outcomes and analytics hit-rate definitions.

Impact: UI display rules and serialized payload validation can drift independently.

Tracked follow-up: centralize shared constants or document one source of truth before adding more mutation-heavy flows under `groupscout-site-2h1`.

### Partial Design Token Mapping

`web/src/design/tokens.js` exports only the token subset needed by current tests. One observed risk is component token references that point to nested token names without a resolver/parity check against `DESIGN.md`.

Impact: future rendered CSS could rely on unresolved token references even while current model tests pass.

Tracked follow-up: add token parity or resolver tests before introducing real CSS rendering under `groupscout-site-2h1`.

### Recursive Credential Guard Boundary

`test/api-boundary.test.js` recursively scans browser-facing `web/src/**/*.js` files and skips `web/src/server`.

Impact: the old file-list risk is reduced, but files placed under `web/src/server` are treated as server-only and are excluded from browser credential checks.

Tracked follow-up: keep browser modules out of `web/src/server`, or make the server/browser boundary explicit when a real bundler exists under `groupscout-site-2h1`.

### Mutation Metadata Is Not A Full Payload Schema

`leadStatus.js` exposes `mutationFields` as UI metadata. The `corrected` action requires `correctionReason` and serializes it, but its `mutationFields` list only calls out `status` and `corrections`.

Impact: treating `mutationFields` as the complete PATCH schema would miss `correctionReason`.

Tracked follow-up: either document `mutationFields` as display metadata only or include all serialized fields in the metadata under `groupscout-site-2h1`.

### Mock Route Parsing

`shell.js` extracts lead IDs from `/leads/{id}` with a raw string slice.

Impact: query strings, trailing slashes, and encoded route params will be treated as literal IDs.

Tracked follow-up: normalize route params when a real router is introduced under `groupscout-site-2h1`.

### Repeated Screen Scaffolding

Screen factories repeat similar state, layout, control, token, loading, empty, and error metadata patterns.

Impact: this is manageable at the current model level, but broad behavior changes could become noisy as more screens are added.

Tracked follow-up: wait for a repeated behavior change before extracting shared helpers; avoid premature framework-like abstractions under `groupscout-site-2h1`.

### Brittle Large-Shape Tests

Some tests assert large object shapes and exact display strings.

Impact: refactors that preserve behavior can still require many assertion updates.

Tracked follow-up: keep high-signal contract assertions, but prefer focused assertions when adding new tests for copy-heavy screen models under `groupscout-site-2h1`.

### Model-Level Test Coverage Only

The current tests assert JavaScript objects and metadata, not browser-rendered UI.

Impact: green tests do not prove keyboard behavior, visual layout, focus handling, or responsive rendering.

Tracked follow-up: add browser/component coverage when a renderer exists under `groupscout-site-kb4`.

### Missing Runtime Contract

The tests use modern Node and web APIs, but `package.json` does not declare an `engines` field.

Impact: developers on older Node versions can see confusing failures.

Tracked follow-up: standardize Node version in docs first, then add an `engines` field when package policy is ready under `groupscout-site-2h1`.

## Backend Repo Documentation Drift

The backend repo was inspected as a source for startup and testing instructions. A few docs appear stale relative to current config:

- Some Docker log examples use service name `app`; current compose service is `groupscout`.
- Some Postgres examples reference generated container names; current compose sets `groupscout_postgres`.
- Some docs reference `docs/API_TESTING.md`, but that file was not present during inspection.
- Email provider docs mention `SENDGRID_API_KEY`, while `.env.example` and config use `RESEND_API_KEY`.

Tracked follow-up: update backend docs in `/mnt/c/Users/alvin/GolandProjects/groupscout` in a separate backend-docs pass under `groupscout-site-2h1`.
