# Agent Instructions

This project uses **bd** (beads) for issue tracking. Run `bd prime` for full workflow context, and use the Beads integration block below as the canonical command reference.

## Known Environment Notes

- This checkout lives on a Windows-mounted WSL path (`/mnt/c/Users/alvin/groupscout-site`). `bd` may warn that `.beads` is mode `0777` and recommend `chmod 700`; `chmod` does not persist on this DrvFs/9p mount without Linux metadata support. Treat that warning as an accepted local limitation for this coordination repo unless the repo is moved to a POSIX filesystem or the WSL mount is reconfigured with metadata support.

## Repository Roles

This repository is the coordination point for GroupScout work. Future feature and bug-fix prompts should start here, then agents should route implementation work to the owning code repository:

- Backend: `/mnt/c/Users/alvin/GolandProjects/groupscout`
- Frontend: `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`
- Centralized backend docs: `backend/`
- Centralized frontend docs: `frontend/`

When work touches backend or frontend code, create or reuse a task branch in that repo before editing where possible. Keep source-repo changes scoped to implementation, tests, and runtime files; markdown project docs now live here unless a repo-local generated file is required.

## Non-Interactive Shell Commands

**ALWAYS use non-interactive flags** with file operations to avoid hanging on confirmation prompts.

Shell commands like `cp`, `mv`, and `rm` may be aliased to include `-i` (interactive) mode on some systems, causing the agent to hang indefinitely waiting for y/n input.

**Use these forms instead:**
```bash
# Force overwrite without prompting
cp -f source dest           # NOT: cp source dest
mv -f source dest           # NOT: mv source dest
rm -f file                  # NOT: rm file

# For recursive operations
rm -rf directory            # NOT: rm -r directory
cp -rf source dest          # NOT: cp -r source dest
```

**Other commands that may prompt:**
- `scp` - use `-o BatchMode=yes` for non-interactive
- `ssh` - use `-o BatchMode=yes` to fail instead of prompting
- `apt-get` - use `-y` flag
- `brew` - use `HOMEBREW_NO_AUTO_UPDATE=1` env var

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
