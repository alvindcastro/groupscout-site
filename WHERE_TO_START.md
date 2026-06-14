# Where To Start

Date: 2026-06-14

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

1. Resolve or explicitly accept `groupscout-site-783`. The `.beads` permission warning appears on every Beads command in this Windows-mounted repo and creates noise during handoffs.
2. Finish or split `groupscout-site-0m0`. It is the only in-progress item and owns current UI checkout drift around admin/session artifacts and Phase 15 hardening.
3. Reconcile `groupscout-site-crz` before trusting old EvalOps or UI smoke claims in backend docs.

After those hygiene/reconciliation tasks, move to runtime correctness:

- `groupscout-site-ei7` - normal `/run` collector and raw persistence warnings.
- `groupscout-site-wda` - Postgres enrichment raw-input foreign-key integration failure.

Then move to the operator UI bridge:

- `groupscout-site-eqm` - backend `/api/*` routes needed by the UI.
- `groupscout-site-29q` - generated frontend API client/types from OpenAPI.
- `groupscout-site-4cv` - sanitized raw audit preview.
- `groupscout-site-kb4` - real browser Phase 15 harness after current-checkout drift is resolved.

## Documentation Anchors

- Active not-done and upgrade order: [backend/docs/planning/NOT_DONE_AND_UPGRADES.md](./backend/docs/planning/NOT_DONE_AND_UPGRADES.md)
- Strategic roadmap: [backend/docs/planning/ROADMAP.md](./backend/docs/planning/ROADMAP.md)
- Frontend current-checkout reconciliation: [frontend/README.md](./frontend/README.md)
- UI implementation history: [frontend/docs/ui-tdd-phase-0-15-implementation.md](./frontend/docs/ui-tdd-phase-0-15-implementation.md)
- Frontend smell refactor prompts: [frontend/docs/code-smell-transformation-prompts.md](./frontend/docs/code-smell-transformation-prompts.md)

## Known Drift To Reconcile

- `frontend/docs/ui-tdd-phase-0-15-implementation.md` records branch-history evidence; use `groupscout-site-0m0` to decide whether to restore missing files or narrow the docs to current behavior.
- `backend/docs/CHANGELOG.md` mentions UI API and pipeline/stat/system work that some planning contracts still mark as not live; verify against the backend source before relying on those claims.
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
