# GroupScout Codex Agents

This plugin is a repo-local Codex skill bundle derived from the existing markdown agent specs in `backend/.claude/agents/` in the backend source repository.

## What it contains

- One Codex skill per existing Claude agent or prompt file.
- A plugin manifest in `.codex-plugin/plugin.json`.
- Per-skill `agents/openai.yaml` files so the skills can be surfaced as named Codex agents.

## Included skills

- `api-integrator`
- `database-architect`
- `enrichment-processor`
- `lux-api-tester`
- `lux-content-writer`
- `lux-lead-classifier`
- `lux-lead-sequence-writer`
- `lux-slack-notifier`
- `lux-status-email-writer`
- `tdd-tester`
- `utility-collector-builder`
- `utility-enrichment-prompt`
- `verification-analyst`

## Notes

- The source of truth for role boundaries remains the backend source repo's `.claude/agents/` specs unless a later plugin migration explicitly supersedes them.
- Marketplace registration is environment-specific. To index this plugin locally, add `plugins/groupscout-agents` to the active Codex marketplace configuration for the workspace.
- Plugin drift review is tracked by `groupscout-site-a1h`: verify Codex marketplace registration, compare plugin skills against backend agent specs, and document Junie/Codex source-of-truth boundaries.
