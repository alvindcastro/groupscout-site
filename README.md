# GroupScout Coordination Repo

This repo is the starting point for GroupScout planning, feature prompts, bug-fix prompts, and documentation.

## Code Repositories

- Backend: `/mnt/c/Users/alvin/GolandProjects/groupscout`
- Frontend: `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`

## Documentation Layout

- `backend/` contains markdown moved from the backend repo, preserving the original relative paths.
- `frontend/` contains markdown moved from the frontend repo, preserving the original relative paths.
- Root `AGENTS.md` and `CLAUDE.md` describe how agents should coordinate work from this repo.

## Agent Workflow

Start from this repo, run `bd ready` to find live work, inspect the centralized docs, then delegate implementation to the backend and frontend repos as needed. The current cross-repo follow-up index lives in `backend/docs/planning/ROADMAP.md` under "Live Beads Follow-Ups"; the dated not-done and upgrade sequencing snapshot lives in `backend/docs/planning/NOT_DONE_AND_UPGRADES.md`. Beads remains the source of truth for active tasks.

Create source-repo branches for implementation work when possible, and keep long-lived project markdown in this coordination repo.
