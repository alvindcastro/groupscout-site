# GroupScout Web App Phase Prompt Pack

> Copy one phase prompt at a time into a coding session. Each phase is intentionally narrow and must begin with failing tests.

## Red-Green-Refactor Checklist

- [x] Read the relevant docs and current code before editing.
- [x] Create or claim a beads issue.
- [x] Write the smallest failing test first.
- [x] Run the focused command and capture the expected red failure.
- [x] Implement only enough to pass.
- [x] Rerun the focused command.
- [x] Run the relevant broader suite.
- [x] Update the phase notes with red evidence, green evidence, changed files, and residual risk.
- [x] Close or update beads issue status.

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

## Phase Prompts

Implementation note, 2026-05-09: the canonical Phase 0-15 sequence in `../UI_TDD_PHASE_PROMPTS.md` has been implemented and documented in [UI TDD Phases 0-15 Implementation Status](./ui-tdd-phase-0-15-implementation.md). This older prompt pack remains as supporting planning context.

### 0. Baseline Reconciliation

- [x] Prompt: reconcile current checkout with Phase 13 UI baseline without product changes.
- [x] Red tests: package scripts, test runner, renderer/runtime metadata, route shell metadata, secret scan.
- [x] Green target: runnable minimal harness or explicitly documented docs-only baseline.
- [x] Verification: focused baseline test plus `git status`.

### 1. Product Contract And UX Guardrails

- [x] Prompt: add tested IA, token, and UX guardrails.
- [x] Red tests: navigation routes, token map, component token names, no browser secrets.
- [x] Green target: shell placeholders, token exports, UX contract docs.
- [x] Verification: focused shell/design tests.

### 2. Backend Compatibility Smoke

- [x] Prompt: classify live `/api/*` compatibility before wiring screens.
- [x] Red tests: `/api/system`, `/api/leads`, `/api/pipeline/runs`, `/api/stats`, `/api/alerts`, `/api/leads/{id}/raw`.
- [x] Green target: compatibility classifier and docs, not backend fixes.
- [x] Verification: classifier tests with live backend or documented stub.

### 3. API Client Contracts

- [x] Prompt: centralize typed client contracts.
- [x] Red tests: query serialization, response adaptation, pagination, mutation validation, error classification, no secret headers.
- [x] Green target: single browser API boundary with feature adapters.
- [x] Verification: API client tests.

### 4. Today Command Center

- [x] Prompt: build read-only priority command center.
- [x] Red tests: priority leads, aging work, verification problems, active alerts, failed jobs, health, route links.
- [x] Green target: Today screen model/component.
- [x] Verification: screen tests and route shell tests.

### 5. Lead Inbox

- [x] Prompt: build dense lead triage.
- [x] Red tests: filters, columns, states, keyboard row activation, responsive modes.
- [x] Green target: inbox screen with mocked client data and query state.
- [x] Verification: inbox tests and token/accessibility assertions.

### 6. Lead Detail Evidence

- [x] Prompt: build evidence-first lead detail.
- [x] Red tests: summary, source evidence, AI enrichment, raw audit link, corrections, activity, responsive layout.
- [x] Green target: read-only detail workspace.
- [x] Verification: detail tests.

### 7. Status, Ownership, Corrections

- [x] Prompt: add state transition model and auditable mutations.
- [x] Red tests: allowed/disallowed transitions, visible actions, mutation payloads, required reasons.
- [x] Green target: transition helpers, action controls, API mutation adapter.
- [x] Verification: state model, detail action, and client mutation tests.

### 8. Verification Queue

- [x] Prompt: add focused source/AI review queue.
- [x] Red tests: trigger classification, filters, actions, raw audit alias, redaction blocked state.
- [x] Green target: verification queue and raw audit client link.
- [x] Verification: queue and raw audit tests.

### 9. Outreach Workspace

- [x] Prompt: add manual outreach draft/log/outcome workflow.
- [x] Red tests: editable fields, validation, copied/sent/logged states, history, no auto-send.
- [x] Green target: outreach workspace and activity integration.
- [x] Verification: outreach screen/client tests.

### 10. Pipeline Monitor

- [x] Prompt: expose pipeline health and queued manual run controls.
- [x] Red tests: run history, collector health, LLM health, delivery failures, queued start.
- [x] Green target: pipeline monitor and client endpoints.
- [x] Verification: pipeline tests.

### 11. Analytics

- [x] Prompt: add explainable demand and lead quality analytics.
- [x] Red tests: distributions, denominators, date ranges, source yield, aging, verification quality.
- [x] Green target: analytics dashboard.
- [x] Verification: stats client and analytics tests.

### 12. Alertd Console

- [x] Prompt: add read-only disruption alert visibility.
- [x] Red tests: alert state, SPS, evidence, room inventory, action history, disabled mutations.
- [x] Green target: read-only alert console.
- [x] Verification: alert tests.

### 13. Auth And Runtime

- [x] Prompt: add same-origin session-safe runtime.
- [x] Red tests: session cookies, base path, dev CORS, deployment readiness, browser secret scan.
- [x] Green target: runtime/session helpers and same-origin API enforcement.
- [x] Verification: deployment/session tests.

### 14. Docker E2E

- [x] Prompt: validate UI with backend Docker stack.
- [x] Red tests: Docker targets, Compose override, healthchecks, ports, backend target, secret-free assets/config.
- [x] Green target: test image, dev server, production static/proxy smoke.
- [x] Verification: Docker config, container tests, `/healthz`, `/`, `/assets/app.js`, `/api/*` classifier.

### 15. Browser UX Hardening

- [x] Prompt: prove the app works in a browser, not just models.
- [x] Red tests: focus, labels, keyboard flows, responsive screenshots, text overflow, no overlapping UI, same-origin API calls.
- [x] Green target: browser-tested primary routes and UX fixes.
- [x] Verification: browser/component tests and screenshot checks at the current deterministic renderer level; real screenshot/pixel checks remain blocked until a browser harness is introduced.
