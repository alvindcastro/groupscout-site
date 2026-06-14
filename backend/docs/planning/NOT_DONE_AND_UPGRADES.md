# Not-Done Work And Upgrade Snapshot

Date: 2026-06-14

Beads is the source of truth for active work. This snapshot is a navigation layer for choosing the next implementation or documentation pass; refresh it with `bd list --status=open` and `bd list --status=in_progress` before starting work.

Current review status after closing the documentation review task: `bd stats` reported 89 issues total, 21 open issues, 1 in-progress issue, 0 blocked issues, and 67 closed issues. The only in-progress task at review time was `groupscout-site-0m0`. `bd blocked` reported no blocked issues.

## Start Here

1. `groupscout-site-783` - resolve or document the `.beads` permission warning on the Windows-mounted coordination repo.
2. `groupscout-site-0m0` - finish or split the stale in-progress UI admin/session and hardening reconciliation.
3. `groupscout-site-crz` - reconcile backend EvalOps and UI smoke artifacts before trusting old roadmap or changelog claims.

The detailed handoff for agents is [WHERE_TO_START.md](../../../WHERE_TO_START.md).

## Highest-Value Unfinished Work

### Coordination And Source-Of-Truth Hygiene

- `groupscout-site-783` - resolve the `.beads` directory permission warning on the Windows-mounted coordination repo.
- `groupscout-site-a1h` - audit Codex plugin skill drift so local agent instructions and plugin docs do not diverge.

### Runtime Correctness And Data Safety

- `groupscout-site-ei7` - investigate normal `/run` collector drift and raw persistence warnings that can hide behind otherwise successful Slack sends.
- `groupscout-site-wda` - investigate the Postgres enrichment raw-input foreign-key integration failure.
- `groupscout-site-crz` - restore or reconcile backend EvalOps and UI Docker smoke artifacts that docs reference but the inspected backend snapshot lacks.

### Operator UI And API Bridge

- `groupscout-site-0m0` - restore or reconcile UI admin session and browser hardening artifacts in the current UI checkout.
- `groupscout-site-eqm` - implement the planned backend-owned operator UI API routes.
- `groupscout-site-29q` - generate frontend API client types from OpenAPI after backend contracts are stable.
- `groupscout-site-4cv` - implement sanitized raw audit preview instead of exposing raw payload bytes in browser state.
- `groupscout-site-kb4` - add a real-browser Phase 15 harness for focus, layout, screenshot, and overlap checks.
- `groupscout-site-3gq` - integrate shared `alertd` state into the alerts API.

### Observability, AI, And Analytics Upgrades

- `groupscout-site-yyj` - add Prometheus metrics, collector freshness, system-health fields, and dashboard documentation.
- `groupscout-site-vud` - implement the LLM provider abstraction.
- `groupscout-site-48g` - add AI-ready SQL and LLM observability runtime.
- `groupscout-site-4b4` - implement Phase 28 analytics and source attribution.

### Lead Workflow And Automation Upgrades

- `groupscout-site-62h` - add Slack lead action buttons and signature verification.
- `groupscout-site-iuc` - implement Phase 18 contact enrichment and budget tiers.
- `groupscout-site-39g` - execute the home deploy path, lock down observability surfaces, configure backups, and run one restore smoke.

### Expansion And Architecture

- `groupscout-site-2h1` - continue remaining backend and frontend smell refactors.
- `groupscout-site-aaj` - plan the next source-expansion collector slice.
- `groupscout-site-1ee` - design multi-property routing support.

## Recommended Upgrade Order

1. **Stabilize documentation and repo state first.** Clear `groupscout-site-783`, `groupscout-site-0m0`, and `groupscout-site-crz` so future agents can trust the repo state and smoke commands.
2. **Fix runtime warnings before expanding scope.** Prioritize `groupscout-site-ei7` and `groupscout-site-wda`; both are correctness risks that can be masked by successful end-to-end runs.
3. **Ship the operator UI contract bridge.** Implement `groupscout-site-eqm`, then generate typed clients with `groupscout-site-29q`, then add raw-preview and browser-hardening work under `groupscout-site-4cv` and `groupscout-site-kb4`.
4. **Upgrade operations visibility.** Finish `groupscout-site-yyj` before relying on dashboards or health views for production decisions.
5. **Improve lead conversion workflows.** Add Slack actions, contact enrichment, analytics, and source attribution through `groupscout-site-62h`, `groupscout-site-iuc`, and `groupscout-site-4b4`.
6. **Harden the AI and deployment platform.** Sequence `groupscout-site-vud`, `groupscout-site-48g`, and `groupscout-site-39g` after the runtime and UI bridge are stable.
7. **Expand sources and routing deliberately.** Use `groupscout-site-aaj` and `groupscout-site-1ee` after the current pipeline, observability, and operator surfaces can absorb more lead volume.

## Documentation Maintenance Rules

- Do not add active markdown TODO lists. File or update Beads issues for active work, then reference the issue IDs from docs.
- When an implementation lands in a source repo, update this coordination repo's markdown in the same session if the behavior, runbook, or roadmap changes.
- Treat old roadmap checkboxes as historical planning context unless an open Beads issue confirms the work is still active.
- When docs mention a command or artifact that is missing from an inspected source checkout, either restore the artifact in the owning repo or document the reconciliation issue ID next to the claim.
- When prompt files name migration numbers, inspect the backend source repo's current migrations before creating new migration files.
