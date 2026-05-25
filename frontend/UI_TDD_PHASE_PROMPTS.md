# GroupScout Web App TDD Phase Prompts

> Planning artifact. Every implementation phase must use strict TDD: write a failing test first, run the narrow red command, implement the smallest change, rerun the narrow command, then run the relevant broader suite.

## Companion Docs

- [Web App Brainstorm](./docs/web-app-brainstorm.md)
- [UI TDD Phases 0-15 Implementation Status](./docs/ui-tdd-phase-0-15-implementation.md)

## Global Rules

- [ ] Use beads for task tracking before code changes.
- [ ] Check current repo state before edits; do not revert user changes.
- [ ] Start every phase by restating goal, workflow, out-of-scope items, and acceptance criteria.
- [ ] Write tests before implementation.
- [ ] Confirm the red failure is for the expected missing behavior.
- [ ] Implement only enough code/docs/runtime behavior to pass.
- [ ] Rerun the narrow test and then the relevant broader suite.
- [ ] Preserve browser-only access through same-origin `/api/*` contracts.
- [ ] Keep secrets server-side: no `API_TOKEN`, provider keys, Slack tokens, Resend keys, database URLs, `OLLAMA_ENDPOINT`, or `UI_SESSION_SECRET` in browser-visible code/assets/config.
- [ ] Use `DESIGN.md` tokens and UX direction: dense operator surfaces, Inter prose, Geist Mono for structured values, hairline borders, compact controls, black primary buttons, mint only for active/confirmation accents.
- [ ] Treat raw source evidence, AI enrichment, reviewer corrections, outreach logs, and status history as distinct audit concepts.
- [ ] Add follow-up beads issues for known gaps instead of hiding them in comments or unchecked assumptions.

## Suggested Phase Order

- [x] Phase 0 - Baseline Reconciliation And Harness
- [x] Phase 1 - Product Contract, IA, And UX Guardrails
- [x] Phase 2 - Backend Compatibility Smoke
- [x] Phase 3 - API Client Contracts
- [x] Phase 4 - Today Command Center
- [x] Phase 5 - Lead Inbox
- [x] Phase 6 - Lead Detail Evidence Workspace
- [x] Phase 7 - Lead Status, Ownership, And Corrections
- [x] Phase 8 - Verification Queue And Raw Audit Review
- [x] Phase 9 - Outreach Workspace And Activity Log
- [x] Phase 10 - Pipeline Monitor And Run Controls
- [x] Phase 11 - Analytics And Demand Signals
- [x] Phase 12 - Alertd Read-Only Console
- [x] Phase 13 - Session/Auth And Same-Origin Runtime
- [x] Phase 14 - Docker Integration And E2E Smoke
- [x] Phase 15 - Browser UX Hardening

## Implementation Status

Completed on 2026-05-09 in the current canonical phase order. See [UI TDD Phases 0-15 Implementation Status](./docs/ui-tdd-phase-0-15-implementation.md) for the phase map, red/green evidence, residual risk, and the restored-baseline decision.

## Parallel Agent Starters

Use these read-only prompts when a phase needs investigation before the main implementation.

### Backend/API Agent

```text
Read-only. Inspect /mnt/c/Users/alvin/GolandProjects/groupscout Markdown, Docker, and API route files relevant to the requested phase. Identify current backend contracts, live route drift, required env vars, Docker service names, and tests that should fail first. Do not edit files. Return concise findings with file paths and line references.
```

### UI Baseline Agent

```text
Read-only. Inspect /mnt/c/Users/alvin/WebstormProjects/groupscout-ui current worktree and origin/main for the requested phase. Identify existing implementation, deleted/local changes, package scripts, tests, runtime files, docs, and likely conflicts. Do not edit files. Return concise findings with file paths and line references.
```

### UX Agent

```text
Read-only. Inspect DESIGN.md and the requested phase. Translate the design system into concrete UI constraints: layout, typography, controls, color usage, state badges, responsive behavior, accessibility, and tests that should fail before implementation. Do not edit files.
```

### Docker/Ops Agent

```text
Read-only. Inspect backend Docker files, UI Docker files if present, Compose ports, networks, env vars, healthchecks, and smoke docs. Identify the smallest Docker/runtime contract tests for the requested phase and any secret exposure risks. Do not edit files.
```

## Phase 0 - Baseline Reconciliation And Harness

### Prompt

```text
You are working in /mnt/c/Users/alvin/WebstormProjects/groupscout-ui. Strictly follow TDD.

Goal: reconcile the current UI checkout with the latest Phase 13 UI baseline before product work begins.

Context:
- The current working tree has local deletions of the previous UI source, tests, package file, docs, and Docker files.
- origin/main contains the latest UI updates: vanilla DOM renderer, node:test suite, static build, product dev server, production static/proxy server, and backend compatibility smoke classification.
- Do not revert user changes unless explicitly directed.

TDD requirements:
1. Add or restore only the smallest test/harness files needed after deciding the baseline strategy.
2. If restoring from origin/main, first add a failing smoke or docs-guardrail test that proves the expected baseline is absent in the current checkout.
3. Run the narrow red command.
4. Restore or recreate the minimal harness to pass.
5. Rerun the narrow command and record the result.

Do not build new product features in this phase.
```

### Tasks

- [ ] Record current `git status` and identify user-owned changes.
- [ ] Decide whether to restore Phase 13 files from `origin/main` or continue from a minimal docs-first baseline.
- [ ] Add a failing baseline smoke for package scripts, route shell, renderer runtime, and secret guardrails.
- [ ] Run the focused red command.
- [ ] Restore or recreate the minimum harness.
- [ ] Rerun focused tests.
- [ ] Update docs with the chosen baseline decision.

## Phase 1 - Product Contract, IA, And UX Guardrails

### Prompt

```text
Strictly follow TDD.

Goal: establish the product contract and visual guardrails for the GroupScout operator workbench.

Context:
- The app is a dense evidence-first operations console, not a CRM replacement or marketing site.
- Use DESIGN.md tokens and UX direction.
- Initial IA: Today, Leads, Verification, Outreach, Pipeline, Analytics, Alerts, Settings.

TDD requirements:
1. Write failing tests for route shell IA, design token availability, reserved navigation labels, and no direct browser references to secrets.
2. Write failing tests or docs assertions for dense operator UX requirements: compact tables, filters, tabs, detail evidence, and responsive variants.
3. Run red.
4. Implement the smallest shell/token metadata needed.
5. Rerun tests.

Do not implement feature screens or live API calls in this phase.
```

### Tasks

- [ ] Test route list and default route.
- [ ] Test `DESIGN.md` token mapping for colors, typography, spacing, radius, and key components.
- [ ] Test no browser-facing secret names.
- [ ] Test shell reserves command center, leads, verification, outreach, pipeline, analytics, alerts, and settings.
- [ ] Implement shell placeholders only after tests fail.
- [ ] Document UX guardrails.

## Phase 2 - Backend Compatibility Smoke

### Prompt

```text
Strictly follow TDD.

Goal: turn backend/UI route assumptions into compatibility smoke tests before wiring screens to live data.

Context:
- Backend docs conflict on whether UI /api/* routes are implemented.
- Backend Compose service is groupscout on 8080.
- Browser-visible code must call same-origin /api/* only.

TDD requirements:
1. Write failing smoke classification tests for /api/system, /api/leads, /api/pipeline/runs, /api/stats, /api/alerts, and /api/leads/{id}/raw.
2. Distinguish proxy failure, backend 404 route drift, auth failure, schema drift, compatible response, and backend error.
3. Run red against a documented live backend or stub.
4. Implement only the classifier/client smoke harness, not route fixes.
5. Rerun focused tests.

Do not hide route drift behind fixtures.
```

### Tasks

- [ ] Inventory backend live routes.
- [ ] Test compatibility classification categories.
- [ ] Test same-origin UI routes map to server-side `http://groupscout:8080`.
- [ ] Test auth/session failures separately from automation `API_TOKEN`.
- [ ] File backend follow-up issues for missing routes.
- [ ] Document current compatible and incompatible endpoints.

## Phase 3 - API Client Contracts

### Prompt

```text
Strictly follow TDD.

Goal: define typed browser API client contracts for UI-modeled /api/* routes.

Context:
- Components should not spread ad hoc fetch calls.
- Contracts must be testable with fixtures before live backend wiring.

TDD requirements:
1. Write failing client tests for leads, lead detail, raw audit, outreach, pipeline runs, stats, alerts, and system health.
2. Test query serialization, response adaptation, error handling, pagination, and forbidden secret headers.
3. Run red.
4. Implement the smallest client boundary.
5. Rerun tests.
```

### Tasks

- [ ] Test `GET /api/leads` filters and pagination.
- [ ] Test `GET /api/leads/{id}` detail shape.
- [ ] Test `PATCH /api/leads/{id}` mutation payload validation.
- [ ] Test `GET /api/leads/{id}/raw` raw audit alias.
- [ ] Test `GET/POST /api/leads/{id}/outreach`.
- [ ] Test `GET/POST /api/pipeline/runs`.
- [ ] Test `GET /api/stats`, `GET /api/alerts`, and `GET /api/system`.
- [ ] Implement one centralized API boundary.

## Phase 4 - Today Command Center

### Prompt

```text
Strictly follow TDD.

Goal: build the first screen: a dense command center for today's operator priorities.

TDD requirements:
1. Write failing tests for priority leads, aging claimed leads, verification problems, active alerts, failed jobs, and system health.
2. Test loading, empty, error, desktop, tablet, and mobile states.
3. Test actions route to owning workspaces instead of duplicating mutations.
4. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test summary counts and generated timestamp.
- [ ] Test high-score new lead rows.
- [ ] Test aging claimed work.
- [ ] Test active alert and failed job cards.
- [ ] Test system health via `/api/system`.
- [ ] Test responsive layout and token use.
- [ ] Implement Today screen.

## Phase 5 - Lead Inbox

### Prompt

```text
Strictly follow TDD.

Goal: build the lead triage inbox.

TDD requirements:
1. Write failing tests for table columns, filter controls, priority ordering, loading/empty/error states, row selection, and detail navigation.
2. Test desktop table, tablet priority table, and mobile lead cards.
3. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test search, status, source, score, owner, property, verification, and date filters.
- [ ] Test score, title, segment, location/property fit, source, crew/duration, timing, status, owner, created, and evidence columns.
- [ ] Test clear filters.
- [ ] Test keyboard row activation.
- [ ] Test mocked and API-backed data paths separately.
- [ ] Implement inbox screen.

## Phase 6 - Lead Detail Evidence Workspace

### Prompt

```text
Strictly follow TDD.

Goal: build the lead detail workspace around evidence and auditability.

TDD requirements:
1. Write failing tests for Summary, Source Evidence, AI Enrichment, Actions, Outreach, and Activity sections.
2. Test original source values, AI values, and reviewer corrections stay visually distinct.
3. Test raw audit links exist without loading raw payload inline by default.
4. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test summary fields.
- [ ] Test source URL, source name, raw audit link, collected timestamp.
- [ ] Test AI rationale, uncertainty, crew, duration, project type, and evidence references.
- [ ] Test activity timeline.
- [ ] Test not found, loading, and error states.
- [ ] Test responsive detail layout.
- [ ] Implement read-only detail workspace.

## Phase 7 - Lead Status, Ownership, And Corrections

### Prompt

```text
Strictly follow TDD.

Goal: implement lead state transitions, ownership actions, and auditable corrections.

TDD requirements:
1. Write failing unit tests for allowed and blocked transitions.
2. Write failing UI tests that only valid actions are shown for the current state.
3. Write failing mutation tests for status, owner, notes, snooze date, and correction reason.
4. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Define and test status model.
- [ ] Test claim, snooze, flag, verify, dismiss, contacted, won, lost, no response, reopen, and correct.
- [ ] Test required notes/reasons.
- [ ] Test invalid transitions never call the API client.
- [ ] Test corrections preserve original source/AI values.
- [ ] Implement action controls and helpers.

## Phase 8 - Verification Queue And Raw Audit Review

### Prompt

```text
Strictly follow TDD.

Goal: build a focused queue for leads that need source or AI review.

TDD requirements:
1. Write failing tests for queue inclusion rules.
2. Test verify, correct, dismiss, and return-to-lead actions.
3. Test raw audit access through /api/leads/{id}/raw.
4. Test redaction-policy behavior before inline raw display.
5. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test missing source/raw audit trigger.
- [ ] Test high score with weak rationale trigger.
- [ ] Test raw/enriched contradiction trigger.
- [ ] Test low-confidence parse trigger.
- [ ] Test manual flag trigger.
- [ ] Test redaction policy as blocked until defined.
- [ ] Implement queue.

## Phase 9 - Outreach Workspace And Activity Log

### Prompt

```text
Strictly follow TDD.

Goal: add manual outreach drafting, logging, and outcome capture.

TDD requirements:
1. Write failing tests for editable draft, contact fields, copied/sent/logged display states, and validation.
2. Write failing client tests for outreach history and logging.
3. Test the UI never auto-sends email.
4. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test channel, contact, message, notes, and outcome fields.
- [ ] Test manual copied/sent/logged states.
- [ ] Test contacted, won, lost, and no-response outcomes.
- [ ] Test activity timeline entries.
- [ ] Test no auto-send behavior.
- [ ] Implement outreach workspace.

## Phase 10 - Pipeline Monitor And Run Controls

### Prompt

```text
Strictly follow TDD.

Goal: expose pipeline health and manual run controls without replacing observability tools.

TDD requirements:
1. Write failing tests for run history, collector counts/failures, LLM health, delivery failures, and queued manual runs.
2. Test manual run creation as accepted async work.
3. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test recent run rows.
- [ ] Test collector collected/skipped/enriched counts.
- [ ] Test LLM provider, latency, and errors.
- [ ] Test Slack/email/webhook delivery failures.
- [ ] Test manual run request payload.
- [ ] Implement pipeline monitor.

## Phase 11 - Analytics And Demand Signals

### Prompt

```text
Strictly follow TDD.

Goal: add basic explainable analytics for lead quality and demand timing.

TDD requirements:
1. Write failing tests for status, source, score, owner, week, property, and segment summaries.
2. Test denominators, date ranges, empty states, and source-yield definitions.
3. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test status distribution.
- [ ] Test source yield and denominators.
- [ ] Test score bands and owner load.
- [ ] Test lead aging and verification quality.
- [ ] Test demand by week/property/segment.
- [ ] Implement analytics dashboard.

## Phase 12 - Alertd Read-Only Console

### Prompt

```text
Strictly follow TDD.

Goal: add a read-only disruption alert console after lead workflows are stable.

TDD requirements:
1. Write failing tests for alert state, SPS, evidence, room inventory, action history, and disabled mutation actions.
2. Test Slack remains the interrupt channel.
3. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test alert list and summary.
- [ ] Test highest SPS and room impact.
- [ ] Test evidence rows.
- [ ] Test action history.
- [ ] Test no acknowledge/resolve/suppress mutation client exists.
- [ ] Implement read-only alert console.

## Phase 13 - Session/Auth And Same-Origin Runtime

### Prompt

```text
Strictly follow TDD.

Goal: make browser access safe enough for operator use.

TDD requirements:
1. Write failing tests for session cookie enforcement, UI base path, dev-only CORS, deployment readiness, and same-origin /api/* routing.
2. Test API_TOKEN is never repurposed as a browser credential.
3. Run red, implement minimum, rerun.
```

### Tasks

- [ ] Test `UI_ENABLED`.
- [ ] Test `UI_BASE_PATH`.
- [ ] Test `UI_SESSION_SECRET` readiness.
- [ ] Test session-cookie authorization for `/api/*`.
- [ ] Test browser bundle secret scans.
- [ ] Implement runtime/session helpers.

## Phase 14 - Docker Integration And E2E Smoke

### Prompt

```text
Strictly follow TDD.

Goal: validate the UI beside the backend Docker stack without leaking secrets.

TDD requirements:
1. Write failing Docker/Compose contract tests for test image, dev product server, production static/proxy server, ports, healthchecks, backend service discovery, and secret-free config.
2. Run red.
3. Implement or restore the smallest Docker files/runtime code needed.
4. Run focused tests, Docker config validation, and smoke checks.
```

### Tasks

- [ ] Test `docker build --target test` contract.
- [ ] Test dev UI service host port defaults to `3001`.
- [ ] Test production smoke can run on `3002`.
- [ ] Test server-side target `http://groupscout:8080`.
- [ ] Test `/healthz`, `/`, `/assets/app.js`, and one `/api/*` classification.
- [ ] Test Compose/static assets are secret-free.
- [ ] Implement Docker integration.

## Phase 15 - Browser UX Hardening

### Prompt

```text
Strictly follow TDD.

Goal: prove the app behaves correctly in a real browser.

TDD requirements:
1. Write failing browser tests for keyboard focus, accessible names, responsive layout, non-overlap, loading states, and same-origin API calls.
2. Add screenshot or pixel checks only after deterministic rendering exists.
3. Run red, implement minimum UX fixes, rerun.
```

### Tasks

- [ ] Test keyboard navigation through shell, filters, rows, tabs, and actions.
- [ ] Test accessible names for icon/action controls.
- [ ] Test desktop, tablet, and mobile layouts.
- [ ] Test no text overflow in buttons, badges, filters, and table/card cells.
- [ ] Test loading and error states do not shift layout incoherently.
- [ ] Test screenshots for primary routes once browser harness exists.
- [ ] Implement UX hardening fixes.
