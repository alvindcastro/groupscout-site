# GroupScout Web App Brainstorm

> Planning artifact. Do not treat this as implementation evidence. The implementation prompts in `UI_TDD_PHASE_PROMPTS.md` require strict red-green-refactor TDD.

## Sources Checked

- `/mnt/c/Users/alvin/GolandProjects/groupscout/README.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/ARCHITECTURE.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/DATA_FLOW.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/API_CONFIG.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/DOCKER.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/TESTING.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/guides/TDD_AI_QUALITY.md`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docs/planning/ui/*`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/Dockerfile`
- `/mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml`
- `DESIGN.md`
- Current UI branch state and Phase 13 UI runtime updates from `origin/main`

## Current Repo Reality

- [x] The previous `feat/master` checkout had local deletions of `package.json`, `web/`, `test/`, Docker files, README, and previous docs.
- [x] The tracked UI baseline was restored on 2026-05-09 after a red Phase 0 reconciliation smoke proved the baseline was absent.
- [x] The restored implementation includes a dependency-free vanilla DOM renderer, `node:test` suite, static build, product dev server, production static/proxy server, backend compatibility smoke classification, API clients, screen models, Docker contracts, and deterministic Phase 15 browser UX hardening metadata.
- [x] Existing unrelated `.idea/*` user changes were preserved.

## Product Idea

GroupScout Web is an operator workbench for hotel sales teams. It turns automated source collection, scoring, enrichment, and alerting into a durable review workflow: what needs attention today, why each lead exists, who owns it, what evidence supports it, what outreach happened, and what outcome came back.

The first useful app is not a marketing site and not a CRM replacement. It is a dense, evidence-first operating console that lets a human review and act on AI-assisted lodging demand signals without losing auditability.

## Primary Workflows

- Start at Today and scan high-priority work: new high-score leads, aging claimed leads, verification problems, failed jobs, active disruption alerts, and system health.
- Triage leads from a dense inbox with filters for status, source, score, owner, property fit, verification state, and created date.
- Open a lead detail workspace that keeps source evidence, raw audit links, AI enrichment, reviewer corrections, status history, and outreach activity visually distinct.
- Claim, snooze, verify, correct, dismiss, mark contacted, mark won/lost, and reopen leads only through tested state transitions.
- Review low-trust or high-impact leads in a verification queue before outreach.
- Draft and log outreach manually without auto-sending email from the UI.
- Monitor pipeline runs, collector health, AI enrichment health, and delivery failures without replacing Grafana or logs.
- Review read-only `alertd` disruption alerts while keeping Slack as the interrupt channel.

## UX Direction From `DESIGN.md`

- Use Inter for UI prose and Geist Mono for code-like values, ids, endpoint paths, source names, and structured evidence.
- Use dense operational layouts: left navigation, compact tables, detail panes, segmented filters, tabs, drawers, and timelines.
- Use white canvas, subtle `surface` backgrounds, and 1px `hairline` borders as the default visual system.
- Use black pill primary buttons for primary commands; reserve mint green for active states, confirmations, and rare accent CTAs.
- Use `rounded.md` for inputs/search/code blocks, `rounded.lg` for cards/panels, and `rounded.full` for buttons, badges, and pill tabs.
- Keep cards for repeated entities or framed tools only. Do not nest cards inside cards.
- Avoid marketing heroes in the operator app. If a welcome/empty state exists, it should still route the operator into work.
- Favor scanability over decoration: numeric summaries, state badges, source chips, evidence rows, and timelines should carry the interface.
- Test responsive behavior explicitly: desktop tables, tablet priority columns, and mobile lead cards/detail stacks.

## Backend And Docker Constraints

- Backend API service is `groupscout` on container port `8080`.
- `alertd` runs separately on `8081`.
- Docker Compose also includes Postgres/pgvector, n8n, Prometheus, Grafana on host `3000`, Loki, Promtail, Ollama, and `ollama-init`.
- UI examples should keep host `3001` for development and `3002` for production-container smoke to avoid Grafana's `3000`.
- UI containers on the backend Docker network should target `http://groupscout:8080` server-side only.
- Browser code must use same-origin relative `/api/*` routes.
- Never expose `API_TOKEN`, provider keys, Slack tokens, Resend keys, database URLs, `OLLAMA_ENDPOINT`, or `UI_SESSION_SECRET` to browser JavaScript, static assets, public config, or Compose output.
- Current docs conflict on whether UI `/api/*` routes are implemented. Compatibility tests must verify live behavior before wiring screens to the backend.

## App Surfaces

- Today: priority work, health summary, failed jobs, active alerts, and links to owning workspaces.
- Leads: triage inbox with dense filters and priority ordering.
- Lead Detail: evidence, enrichment, actions, outreach, status history, and corrections.
- Verification: queue for uncertain, contradicted, manually flagged, or weakly sourced leads.
- Outreach: manual draft, contact details, attempt log, and outcomes.
- Pipeline: run history, queued/manual run controls, collector/LLM/delivery health.
- Analytics: source yield, status distribution, aging, verification quality, owner load, demand by week/property/segment.
- Alerts: read-only `alertd` console for disruption demand signals.
- Settings: only after auth, session, properties, and operator configuration have a tested contract.

## Tracked Product Decisions

- Verification is modeled separately from lead status in the UI screen models; live backend route/state alignment remains tracked by `groupscout-site-eqm` and generated client/type follow-through by `groupscout-site-29q`.
- Operator permissions for claim, verify, correct, dismiss, reopen, and won/lost actions remain part of the admin/session reconciliation tracked by `groupscout-site-0m0`.
- Reviewer corrections live alongside original source and AI values in the UI model; backend contract alignment remains tracked by `groupscout-site-eqm` and `groupscout-site-29q`.
- The documented browser session-cookie path is the current planned auth direction; restoring or reconciling the inspected checkout is tracked by `groupscout-site-0m0`.
- Backend-owned `/api/*` route implementation is tracked by `groupscout-site-eqm`; generated frontend client/types are tracked by `groupscout-site-29q`.
- Sanitized raw audit preview/redaction is tracked by `groupscout-site-4cv`.
- Outreach outcome source-of-truth and backend persistence belong with the planned operator UI API route work in `groupscout-site-eqm`.
- Analytics denominator and freshness definitions are tracked by `groupscout-site-yyj`.
