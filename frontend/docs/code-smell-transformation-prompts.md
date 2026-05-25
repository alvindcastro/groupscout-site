# Code Smell Transformation TDD Prompts

> Planning artifact only. Do not implement code from this file in the current planning pass.
> Every future phase must follow strict TDD: write failing tests first, run them, implement the smallest change, rerun tests, then refactor.

This prompt pack turns the current code-smell watchlist into implementation-ready refactor phases. Use it with [code-smells.md](./code-smells.md), [testing.md](./testing.md), and `UI_TDD_PHASE_PROMPTS.md`.

## Global Rules For Every Smell Phase

- Restate the smell, affected files, user-facing risk, and acceptance criteria before editing code.
- Write characterization tests before moving behavior.
- Run the targeted test command and confirm the new or changed tests fail for the expected reason when behavior is missing.
- Implement the smallest refactor that preserves existing behavior.
- Rerun targeted tests and `npm test`.
- Refactor only after tests pass.
- Keep browser requests behind same-origin `/api/*` contracts.
- Keep `API_TOKEN` out of browser-facing JavaScript.
- Preserve existing screen-model behavior unless the phase explicitly changes it.
- Update docs and changelog in the same phase when contracts, commands, or source-of-truth decisions change.

## Smell Phase H0 - Baseline Characterization

### Prompt

```text
You are working in the GroupScout UI repo. Strictly follow TDD.

Goal: create a stable characterization baseline before transforming code smells.

Context:
- Read docs/code-smells.md.
- Read docs/testing.md.
- Read UI_TDD_PHASE_PROMPTS.md.
- The repo currently uses model-level JavaScript tests with Node's built-in node:test runner.

TDD requirements:
1. Identify the smallest existing tests that cover the behavior touched by the next smell phase.
2. Add or tighten characterization tests before moving code.
3. Run the targeted tests and confirm the red state only when the characterization is intentionally missing.
4. Do not refactor production code in this baseline phase unless a failing characterization test requires a minimal fixture or helper.
5. Record the targeted commands in the phase notes.

Do not start structural refactors until the baseline is explicit.
```

### Tasks

- [x] List affected files for the next smell phase.
- [x] Map each affected behavior to an existing test file.
- [x] Add missing characterization assertions for public exports, payload shapes, route outputs, and policy metadata.
- [x] Run the focused tests.
- [x] Run `npm test`.
- [x] Document the baseline commands and result.

### Acceptance Criteria

- [x] The next refactor has tests that fail for real behavior changes, not file layout changes.
- [x] Existing public contracts are named before implementation starts.
- [x] No broad refactor has started yet.

### Implementation Notes

- Baseline notes live in [smell-h0-api-client-characterization.md](./smell-h0-api-client-characterization.md).
- H0 identified H1 as the next smell phase: split `web/src/api/client.js` while preserving the `createApiClient(...)` facade and centralized same-origin `/api/*` transport guard.
- Added characterization coverage for public exports, client method surface, invalid route pre-fetch rejection, request defaults, encoded lead-scoped routes, payload defaults, adapter defaults, and read-only policy metadata.
- Focused run on 2026-05-09: `node --test test/api-boundary.test.js test/lead-inbox-client.test.js test/lead-status-mutation-client.test.js test/raw-audit-client.test.js test/outreach-client.test.js test/pipeline-client.test.js test/stats-client.test.js test/alert-client.test.js test/system-client.test.js` passed.
- Full run on 2026-05-09: `npm test` passed all 22 test files.
- No production refactor started in H0.

## Smell Phase H1 - Split The Growing API Client

### Prompt

```text
Strictly follow TDD.

Goal: reduce web/src/api/client.js from a single feature-contract module into focused API adapters while preserving one shared same-origin transport guard.

Context:
- web/src/api/client.js currently owns transport rules, path building, payload normalization, response adaptation, and feature-specific contracts.
- The same-origin /api/* guard is security-sensitive and must remain centralized.

TDD requirements:
1. Write or tighten tests that prove absolute URLs and non-/api paths are rejected before fetch.
2. Write characterization tests for each existing client method and adapter shape.
3. Run tests and confirm failures only for missing split-module exports or characterization gaps.
4. Move feature adapters behind focused modules while preserving createApiClient(...) as the stable entry point.
5. Rerun api-boundary tests, all affected client tests, and npm test.

Do not change endpoint paths, payload names, or response adaptation semantics in this phase.
```

### Tasks

- [x] Characterize shared transport behavior in `test/api-boundary.test.js`.
- [x] Characterize lead, raw audit, outreach, pipeline, stats, alerts, and system client methods.
- [x] Define the target module ownership before moving code.
- [x] Keep `createApiClient(...)` backward compatible.
- [x] Move feature-specific adapters one group at a time.
- [x] Rerun focused client tests after each group.
- [x] Update developer docs with the new API module map.

### Acceptance Criteria

- [x] Same-origin transport rules are still centralized and test-covered.
- [x] Feature adapters are easier to review independently.
- [x] Existing callers still use `createApiClient(...)`.
- [x] Full suite passes after the split.

### Implementation Notes

- H1 split `web/src/api/client.js` into the stable facade plus focused modules: `transport.js`, `shared.js`, `leads.js`, `rawAudit.js`, `outreach.js`, `pipeline.js`, `stats.js`, `alerts.js`, and `system.js`.
- `transport.js` owns the same-origin `/api/*` guard, JSON defaults, session credentials, non-2xx handling, and non-JSON success behavior.
- `client.js` remains the public import path and still exports `API_BASE_PATH`, `DEFAULT_LEAD_INBOX_SORT`, `LEAD_INBOX_ITEM_FIELDS`, and `createApiClient(...)`.
- Added an API split ownership assertion to `test/api-boundary.test.js`.
- Focused run on 2026-05-09: `node --test test/api-boundary.test.js test/lead-inbox-client.test.js test/lead-status-mutation-client.test.js test/raw-audit-client.test.js test/outreach-client.test.js test/pipeline-client.test.js test/stats-client.test.js test/alert-client.test.js test/system-client.test.js` passed.
- Full run on 2026-05-09: `npm test` passed all 22 test files.
- H2 is now the next active smell phase.

## Smell Phase H2 - Isolate Mock Fixtures From Runtime Factories

Open H2-H11 smell work is tracked by `groupscout-site-2h1`; the bullets in this file are implementation prompts and acceptance criteria, not standalone task tracking.

### Prompt

```text
Strictly follow TDD.

Goal: separate demo/test fixture data from production screen factories.

Context:
- Screen modules currently export and default to mocked leads, detail evidence, analytics stats, alerts, and Today summaries.
- Live backend wiring will be safer if fixtures are explicit test/demo inputs instead of runtime defaults.

TDD requirements:
1. Write characterization tests for each screen factory with explicit data inputs.
2. Write failing tests that production factory calls do not silently depend on demo fixture exports.
3. Move fixtures to a test/demo fixture boundary after the red state is proven.
4. Rerun screen tests and app-shell tests.

Do not introduce live backend calls or a renderer in this phase.
```

### Tasks

- Inventory all exported mocked data from `web/src/app/*`.
- Add explicit-input tests for Lead Inbox, Lead Detail, Verification, Outreach, Pipeline, Analytics, Alerts, and Today.
- Define where demo fixtures live.
- Move fixtures without changing screen model output.
- Keep tests readable after fixture relocation.
- Update nice-to-knows and developer guide with the fixture boundary.

### Acceptance Criteria

- Production screen factories can be reasoned about separately from demo data.
- Tests still have clear fixture inputs.
- No live data behavior is introduced.

## Smell Phase H3 - Centralize Duplicated Domain Constants

### Prompt

```text
Strictly follow TDD.

Goal: make shared domain constants have one source of truth.

Context:
- Some rules exist in both app model modules and the API client layer, including outreach outcomes and analytics hit-rate definitions.
- Drift here can break serialized payloads or metric explanations.

TDD requirements:
1. Write tests that import constants from the intended source of truth.
2. Write adapter and screen tests that prove the constants are reused, not redefined.
3. Run tests and confirm failures where duplicate local constants still exist.
4. Move constants behind a focused domain module.
5. Rerun affected app, client, and analytics tests.

Do not change business definitions unless the product docs are updated in the same phase.
```

### Tasks

- Identify duplicated status, outcome, and metric-definition constants.
- Choose one domain module for shared UI constants.
- Test outreach outcomes from screen and API serialization paths.
- Test analytics hit-rate definitions from dashboard and stats paths.
- Replace local copies after tests fail.
- Update docs that name the source of truth.

### Acceptance Criteria

- Shared UI constants have one tested owner.
- Payload validation and display labels cannot drift silently.
- Existing metric definitions remain unchanged unless explicitly documented.

## Smell Phase H4 - Add Design Token Parity Or Resolution

### Prompt

```text
Strictly follow TDD.

Goal: make design token references verifiable beyond the current partial export checks.

Context:
- web/src/design/tokens.js exports only the token subset needed by current tests.
- Screen models can reference nested token names without a resolver proving they map back to DESIGN.md.

TDD requirements:
1. Write failing tests for token names used by current screen models.
2. Add a resolver or parity check that catches unresolved token references.
3. Run tests and confirm unresolved or missing token references fail clearly.
4. Implement the smallest token validation surface needed.
5. Rerun design-token tests and screen tests.

Do not redesign the palette, typography, spacing, or component token names in this phase.
```

### Tasks

- Inventory component token references from screen models.
- Add tests for unresolved token references.
- Decide whether parity is checked against `DESIGN.md` text or a normalized token map.
- Implement the smallest resolver/checker.
- Update testing docs with the new token command.
- Keep visual design unchanged.

### Acceptance Criteria

- Current token references are resolvable.
- Missing token names fail in focused tests.
- `DESIGN.md` remains the design source contract.

## Smell Phase H5 - Make The Browser/Server Boundary Explicit

### Prompt

```text
Strictly follow TDD.

Goal: reduce risk from the recursive credential guard excluding web/src/server.

Context:
- The credential guard scans browser-facing web/src/**/*.js and skips web/src/server.
- Files placed under web/src/server are treated as server-only.

TDD requirements:
1. Write tests that document which source paths are considered browser-facing.
2. Write failing tests for accidental browser module placement under server-only paths if a manifest or allow-list is introduced.
3. Implement the smallest explicit boundary check.
4. Rerun api-boundary and session-deployment tests.

Do not weaken the existing API_TOKEN guard.
```

### Tasks

- Document current browser-facing and server-only path rules.
- Add tests for allowed server-only files.
- Add tests for browser modules staying outside `web/src/server`.
- Introduce a manifest, naming convention, or scanner helper only after tests require it.
- Update troubleshooting docs with the boundary rule.

### Acceptance Criteria

- Developers can tell which files are browser-bundled.
- Automation credentials remain excluded from browser code.
- The guard fails clearly when files cross the boundary.

## Smell Phase H6 - Clarify Mutation Metadata Versus Payload Schema

### Prompt

```text
Strictly follow TDD.

Goal: prevent UI mutation metadata from being mistaken for a complete PATCH payload schema.

Context:
- leadStatus.js exposes mutationFields as UI metadata.
- The corrected action serializes correctionReason even though mutationFields does not list every serialized field.

TDD requirements:
1. Write tests that distinguish display metadata from serialized payload requirements.
2. Write failing tests for corrected payload fields, including correctionReason.
3. Either rename/document mutationFields as display metadata or expand schema metadata to include all serialized fields.
4. Rerun lead-status and mutation-client tests.

Do not alter valid status transitions in this phase.
```

### Tasks

- Characterize every action's display metadata.
- Characterize every action's serialized payload fields.
- Decide whether to rename metadata or add full schema metadata.
- Update tests before implementation.
- Update phase 4 docs and developer guide.

### Acceptance Criteria

- Display metadata and payload schema are unambiguous.
- Correction reason remains required and serialized.
- Status transition behavior is unchanged.

## Smell Phase H7 - Normalize Route Parameter Parsing

### Prompt

```text
Strictly follow TDD.

Goal: replace raw string slicing for /leads/{id} with normalized route parameter handling.

Context:
- shell.js extracts lead IDs with a raw string slice.
- Query strings, trailing slashes, and encoded route params can be treated as literal IDs.

TDD requirements:
1. Write failing route tests for /leads/{id}, /leads/{id}/, query strings, and encoded IDs.
2. Confirm the current parser fails the new edge cases.
3. Implement the smallest route normalization helper.
4. Rerun app-shell and lead-detail tests.

Do not introduce a full router unless tests prove it is needed.
```

### Tasks

- Add route tests for trailing slash behavior.
- Add route tests for query string stripping.
- Add route tests for encoded ID decoding or explicit rejection.
- Implement a small route parameter helper after red.
- Document route expectations in developer docs.

### Acceptance Criteria

- Lead detail routes parse stable IDs.
- Existing navigation sections still resolve correctly.
- The shell remains lightweight.

## Smell Phase H8 - Reduce Repeated Screen Scaffolding

### Prompt

```text
Strictly follow TDD.

Goal: extract only the screen scaffolding that has proven duplication across state, layout, control, token, loading, empty, and error metadata.

Context:
- Screen factories repeat similar metadata patterns.
- The duplication is manageable today, so extraction should happen only around repeated behavior changes.

TDD requirements:
1. Write characterization tests for one repeated behavior across at least three screens.
2. Confirm tests fail before adding a shared helper if the shared behavior is missing or inconsistent.
3. Extract a narrow helper for that behavior only.
4. Rerun every screen test that uses the helper.

Do not create a framework-like abstraction or rewrite all screens at once.
```

### Tasks

- Pick one repeated behavior with real maintenance cost.
- Characterize that behavior in at least three screen tests.
- Extract a narrow helper after red.
- Migrate one screen first, then the remaining affected screens.
- Stop when the tested duplication is removed.

### Acceptance Criteria

- Repeated behavior has one tested helper.
- Screen-specific data remains easy to read.
- Refactor scope stays narrow.

## Smell Phase H9 - Replace Brittle Large-Shape Assertions

### Prompt

```text
Strictly follow TDD.

Goal: make tests assert high-signal contracts instead of large object snapshots and exact copy-heavy shapes.

Context:
- Some tests assert large objects and exact display strings.
- Behavior-preserving refactors can become noisy.

TDD requirements:
1. Identify brittle assertions in the tests touched by a real change.
2. Replace them with focused assertions that still prove behavior.
3. Run tests before and after each assertion rewrite.
4. Keep at least one contract-level assertion for required shape boundaries.

Do not weaken coverage or remove assertions only for convenience.
```

### Tasks

- Inventory large-shape assertions in the target test file.
- Decide which fields are contract-critical.
- Rewrite one assertion group at a time.
- Run the focused test after each rewrite.
- Keep copy assertions only where copy is product behavior.

### Acceptance Criteria

- Tests still fail for behavior regressions.
- Tests stop failing for irrelevant object-order or copy-adjacent refactors.
- Required API and screen contract fields remain covered.

## Smell Phase H10 - Declare Runtime And Browser Coverage Readiness

### Prompt

```text
Strictly follow TDD.

Goal: make runtime expectations and future browser/component coverage explicit.

Context:
- Tests rely on modern Node and web APIs, but package.json does not declare an engines field.
- Current coverage is model-level only.

TDD requirements:
1. Write or update tests/docs checks that make the expected Node runtime visible.
2. If package policy allows, add an engines field only after docs/tests define the runtime expectation.
3. Document the first browser/component coverage targets before adding a renderer.
4. Rerun npm test.

Do not add a browser framework, bundler, or dependency install step in this phase unless separately approved.
```

### Tasks

- Confirm the minimum Node version from current tests.
- Document the runtime expectation in developer and testing docs.
- Decide whether `package.json` should declare `engines`.
- List the first browser/component tests to add when a renderer exists.
- Keep the no-build harness intact.

### Acceptance Criteria

- Developers know the supported Node runtime.
- Browser coverage gaps are documented and routed to the owning bead.
- No new framework dependency is introduced by accident.

## Smell Phase H11 - Backend Documentation Drift Follow-Up

### Prompt

```text
Strictly follow TDD where tests exist, and otherwise use docs verification commands.

Goal: clean backend documentation drift in the sibling backend repo without changing UI behavior.

Context:
- docs/code-smells.md records backend doc drift from the sibling Go repo.
- Current UI backend notes are mirrors, not the source of truth.

TDD requirements:
1. Verify backend commands and file paths from the backend repo before editing docs.
2. Run the smallest relevant backend doc/test command available.
3. Update backend docs in the backend repo, or update UI mirror docs only if the backend source of truth changed first.
4. Record commands and results.

Do not change backend runtime code during a docs-drift pass unless explicitly requested.
```

### Tasks

- Verify current backend compose service names.
- Verify current Postgres container and connection examples.
- Verify email provider environment variable names.
- Verify whether referenced backend docs exist.
- Update source-of-truth backend docs in a separate backend-docs pass.
- Refresh UI mirror docs only after backend docs are current.

### Acceptance Criteria

- UI docs no longer amplify stale backend instructions.
- Backend docs remain the source of truth.
- No UI source code changes are made.

## Suggested Housekeeping Phase Order

- [x] H0 - Baseline Characterization
- [x] H1 - Split The Growing API Client
- H2 - Isolate Mock Fixtures From Runtime Factories (`groupscout-site-2h1`)
- H3 - Centralize Duplicated Domain Constants (`groupscout-site-2h1`)
- H4 - Add Design Token Parity Or Resolution (`groupscout-site-2h1`)
- H5 - Make The Browser/Server Boundary Explicit (`groupscout-site-2h1`)
- H6 - Clarify Mutation Metadata Versus Payload Schema (`groupscout-site-2h1`)
- H7 - Normalize Route Parameter Parsing (`groupscout-site-2h1`)
- H8 - Reduce Repeated Screen Scaffolding (`groupscout-site-2h1`)
- H9 - Replace Brittle Large-Shape Assertions (`groupscout-site-2h1`)
- H10 - Declare Runtime And Browser Coverage Readiness (`groupscout-site-2h1`)
- H11 - Backend Documentation Drift Follow-Up (`groupscout-site-2h1`)
