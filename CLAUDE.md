# Project Instructions for AI Agents

This file provides instructions and context for AI coding agents working on this project.

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session that changed tracked files or Beads state**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds. For strictly read-only reviews with no tracked-file or Beads changes, report that no push is required and include the clean `git status` result.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->


## Build & Test

This coordination repo has no root build. Run quality gates from the owning source repository when code changes:

- Backend source: `/mnt/c/Users/alvin/GolandProjects/groupscout`
- Frontend source: `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`

For markdown-only changes in this repo, run focused checks such as `git diff --check` and relevant `rg` audits for task-tracking drift.

## Architecture Overview

This repository coordinates the GroupScout backend and frontend workspaces.

- Backend code lives at `/mnt/c/Users/alvin/GolandProjects/groupscout`.
- Frontend code lives at `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.
- Backend markdown documentation has been centralized under `backend/`.
- Frontend markdown documentation has been centralized under `frontend/`.

## Conventions & Patterns

Start new feature and bug-fix work here, then route code changes to the owning backend or frontend repository. Prefer creating a task branch in the source repository before editing. Keep project markdown in this coordination repo unless a local generated artifact is required by tooling.
