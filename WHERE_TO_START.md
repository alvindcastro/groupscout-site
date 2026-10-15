# Where To Start

Date: 2026-07-01

Beads is the source of truth for active work. This file is a short handoff map for agents and humans deciding which GroupScout task to pick next.

## First Commands

Run these from this coordination repo:

```bash
bd prime
bd ready
bd list --status=in_progress
git status --short --branch
```

If implementation touches source code, route it to the owning repo:

- Backend source: `/mnt/c/Users/alvin/GolandProjects/groupscout`
- Frontend source: `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`
- Long-lived project docs: this coordination repo

Create or reuse a task branch in the owning source repo before implementation work where possible.

## Current Starting Point

Always run `bd ready` first; Beads is authoritative when this handoff ages. As of 2026-07-01, the next starting points are:

1. `groupscout-site-7ak` - verify a sending domain in Resend so lead emails can reach non-owner recipients.

Recently completed source-of-truth cleanup:

- `groupscout-site-wda` - Postgres enrichment raw-input FK integration failure fixed: root cause was cross-package test concurrency (shared `TEST_POSTGRES_URL` DB), now serialized with a Postgres advisory lock (backend branch `task/enrichment-integration-schema-isolation`; merge follow-up `groupscout-site-g95`).
- `groupscout-site-8bp` - VCC `/events` selector drift fixed: collector now matches the live `.event-item` anchor markup and extracts all events (backend branch `task/vcc-events-selectors`).
- `groupscout-site-crz` - backend EvalOps and UI smoke artifacts restored/reconciled.
- `groupscout-site-ei7` - normal `/run` collector drift and raw persistence warnings fixed.
- `groupscout-site-783` - `.beads` permission warning documented as an accepted Windows-mounted checkout limitation.
- `groupscout-site-lkr` - Podman CLI, Compose provider, backend stack, UI image tests, and backend-owned UI E2E smoke validated.

Infrastructure as Code (epic `groupscout-site-6ah`, plan in [INFRASTRUCTURE_AS_CODE.md](./backend/docs/planning/INFRASTRUCTURE_AS_CODE.md)):

- ~~`groupscout-site-6ah.2`~~ - Phase 1 Compose and image hardening is done (2026-10-14): verified on Podman and merged to backend `main`.
- ~~`groupscout-site-6ah.3`~~ - Phase 2 app config as code is done (2026-10-14). The n8n workflow bootstrap, Prometheus scrape config, Grafana provisioning and Alloy are merged to backend `main`.
- ~~`groupscout-site-6ah.4`~~ - Phase 3 secrets as code is done (2026-10-14). The live `.env` renders from SOPS-encrypted `deploy/secrets/prod.sops.env`. The owner still needs to back up the age key (`groupscout-site-6ah.13`, P1).
- `groupscout-site-6ah.5` - Phase 4 CI is merged (2026-10-15). Every backend `main` commit now publishes `ghcr.io/alvindcastro/groupscout:<sha>`. The owner still needs to set up branch protection and the Renovate app, and decide on UI `main` (`groupscout-site-6ah.14`, [CI.md](./backend/docs/guides/CI.md)).
- `groupscout-site-6ah.10` - Phase 9 monitoring is built and running live (2026-10-15): a cadence dead man's switch, ops metrics and Grafana alerts. The owner merges backend PR #15 and sets up Healthchecks.io and the ops Slack webhook (`groupscout-site-6ah.16`, [MONITORING.md](./backend/docs/guides/MONITORING.md)).
- `groupscout-site-6ah.11` - Phase 10 is built (backend PR #16, 2026-10-15). The home PC's `PodmanMachineAutostart` task now runs `deploy/windows/podman-autostart.ps1`, which also starts the monitoring containers. A new PC needs `winget configure` plus `setup-dev.ps1` ([PODMAN_MIGRATION.md](./backend/docs/guides/PODMAN_MIGRATION.md#machine-autostart-on-windows)).
- Next: `groupscout-site-sfi` (P1, code guards so integration tests can't hit the live database) and `groupscout-site-q74` (alertd has not run since the Podman migration). Phases 5–8 wait on the Phase 0 decisions (`groupscout-site-6ah.1`).
- `groupscout-site-6ah.1` - Phase 0 decisions D1–D11 (deploy target, tooling) gate Phases 5–8.

Then move to the operator UI bridge:

- `groupscout-site-eqm` - backend `/api/*` routes needed by the UI.
- `groupscout-site-1x9` - frontend `/admin/login` route/client after backend auth/session routes land.
- `groupscout-site-29q` - generated frontend API client/types from OpenAPI.
- `groupscout-site-4cv` - sanitized raw audit preview.
- `groupscout-site-kb4` - real browser Phase 15 harness.

## Documentation Anchors

- Infrastructure as Code plan and phase scope: [backend/docs/planning/INFRASTRUCTURE_AS_CODE.md](./backend/docs/planning/INFRASTRUCTURE_AS_CODE.md)
- Compose layout, profiles and pinned images: [backend/docs/guides/DOCKER.md](./backend/docs/guides/DOCKER.md#compose-layout)
- Active not-done and upgrade order: [backend/docs/planning/NOT_DONE_AND_UPGRADES.md](./backend/docs/planning/NOT_DONE_AND_UPGRADES.md)
- Strategic roadmap: [backend/docs/planning/ROADMAP.md](./backend/docs/planning/ROADMAP.md)
- Podman migration runbook: [backend/docs/guides/PODMAN_MIGRATION.md](./backend/docs/guides/PODMAN_MIGRATION.md)
- Secrets (SOPS + age), rendering `.env`, rotation and secret scanning: [backend/docs/guides/SECRETS.md](./backend/docs/guides/SECRETS.md)
- CI, GHCR images, the vulnerability gate and Renovate: [backend/docs/guides/CI.md](./backend/docs/guides/CI.md)
- Monitoring, alerts and the cadence dead man's switch: [backend/docs/guides/MONITORING.md](./backend/docs/guides/MONITORING.md)
- Frontend current-checkout reconciliation: [frontend/README.md](./frontend/README.md)
- UI implementation history: [frontend/docs/ui-tdd-phase-0-15-implementation.md](./frontend/docs/ui-tdd-phase-0-15-implementation.md)
- Frontend smell refactor prompts: [frontend/docs/code-smell-transformation-prompts.md](./frontend/docs/code-smell-transformation-prompts.md)

## Known Drift To Reconcile

- `frontend/docs/ui-tdd-phase-0-15-implementation.md` records branch-history evidence; current checkout now has `getLead`, production session gating, and deterministic Phase 15 hardening again, while `/admin/login` remains planned under `groupscout-site-1x9`.
- `backend/docs/CHANGELOG.md` mentions UI API and pipeline/stat/system work that some planning contracts still mark as not live; verify against the backend source before relying on those claims.
- The `.beads` permission warning on this Windows-mounted coordination checkout is accepted under `groupscout-site-783`; `chmod 700 .beads` does not persist on the current DrvFs/9p mount without metadata support.
- Historical roadmap files can conflict with the Beads order. Prefer Beads plus `NOT_DONE_AND_UPGRADES.md` over older "next" lists.
- Migration-number prompt files are guidance only. Inspect current backend migrations before creating new migration filenames.

## Session Close

If tracked files or Beads state changed, do not stop until the normal close sequence succeeds:

```bash
git status
git add <files>
git commit -m "..."
git pull --rebase
bd dolt push
git push
git status
```
