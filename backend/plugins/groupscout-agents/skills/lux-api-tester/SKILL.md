---
name: lux-api-tester
description: Use when creating, updating, or verifying Bruno API collection requests for LUX MVP workflows A, B, or C. Derived from .claude/agents/lux-api-tester.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/lux-api-tester.md"
---

# LUX API Tester

You create and maintain Bruno API collection requests for LUX MVP n8n workflows.

## Historical collection location

The original LUX Bruno folders and `docs/mvps/` payload files are historical/migrated in this coordinator docs tree. Use this skill and the backend source repo's `.claude/agents/lux-api-tester.md` as the current role reference unless the active workflow branch restores those files.

- `api/bruno/environments/Local.bru`
- `api/bruno/lux-mvp-a/`
- `api/bruno/lux-mvp-b/`
- `api/bruno/lux-mvp-c/`

## Environment variables

- `base_url`: `http://localhost:8080`
- `alertd_url`: `http://localhost:8081`
- `n8n_url`: `http://localhost:5678`
- `api_token`: `changeme`

n8n webhooks do not use bearer auth. GroupScout server endpoints do.

## Bruno request shape

```bru
meta {
  name: "Human-readable name"
  type: http
  seq: 1
}

post {
  url: {{n8n_url}}/webhook/<webhook-path>
  body: json
}

body:json {
  { <payload> }
}
```

Use bearer auth for GroupScout server endpoints that require `{{api_token}}`.

## LUX MVP webhook URLs

- MVP-A: `/webhook/lux-status-email`
- MVP-B: `/webhook/lux-lead-followup`
- MVP-C: `/webhook/lux-linkedin-post`

For test mode, replace `/webhook/` with `/webhook-test/`.

## Rules

- Folder names match the MVP label.
- `seq` numbering restarts per folder.
- Keep Bruno bodies synced to the active migrated payload source for the workflow.
- If you change a Bruno request, update the corresponding setup guide.
