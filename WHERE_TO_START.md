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

1. `groupscout-site-lkr` - install Podman CLI and run the migration smoke sequence.
2. `groupscout-site-7ak` - verify a sending domain in Resend so lead emails can reach non-owner recipients.
3. `groupscout-site-8bp` - verify VCC `/events` selectors after the URL drift fix.
4. `groupscout-site-wda` - investigate the Postgres enrichment raw-input foreign-key integration failure.

Recently completed source-of-truth cleanup:

- `groupscout-site-crz` - backend EvalOps and UI smoke artifacts restored/reconciled.
- `groupscout-site-ei7` - normal `/run` collector drift and raw persistence warnings fixed.
- `groupscout-site-783` - `.beads` permission warning documented as an accepted Windows-mounted checkout limitation.

Then move to the operator UI bridge:

- `groupscout-site-eqm` - backend `/api/*` routes needed by the UI.
- `groupscout-site-1x9` - frontend `/admin/login` route/client after backend auth/session routes land.
- `groupscout-site-29q` - generated frontend API client/types from OpenAPI.
- `groupscout-site-4cv` - sanitized raw audit preview.
- `groupscout-site-kb4` - real browser Phase 15 harness.

## Documentation Anchors

- Active not-done and upgrade order: [backend/docs/planning/NOT_DONE_AND_UPGRADES.md](./backend/docs/planning/NOT_DONE_AND_UPGRADES.md)
- Strategic roadmap: [backend/docs/planning/ROADMAP.md](./backend/docs/planning/ROADMAP.md)
- Podman migration runbook: [backend/docs/guides/PODMAN_MIGRATION.md](./backend/docs/guides/PODMAN_MIGRATION.md)
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
