# GroupScout Codex Agents

This plugin is a repo-local Codex skill bundle derived from the existing markdown agent specs in `.claude/agents/`.

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

- The source of truth for the role boundaries remains `.claude/agents/`.
- This repo's `.agents/` directory is currently not writable, so I did not add a marketplace entry automatically.
- If you want this plugin indexed in a local Codex marketplace, add `plugins/groupscout-agents` to `.agents/plugins/marketplace.json` once that directory is writable.
