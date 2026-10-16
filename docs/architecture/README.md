# GroupScout Architecture Diagrams

Draft diagrams of GroupScout as it runs on 2026-10-16: backend image `9c69038` (`task/iac-phase9-monitoring` merged with `main` at `cd063e0`), the home Podman stack, and the IaC work through Phase 10.

Every diagram is Mermaid inside Markdown, so it renders on GitHub and in the VS Code, JetBrains and Obsidian previews. Each file states what is live, what is built but not merged, and what is planned. Those three states change quickly, so check the date at the top of each file.

| # | File | Answers |
|---|---|---|
| 1 | [System context](./01-system-context.md) | Who uses GroupScout, and which outside systems it reads from and writes to |
| 2 | [End-to-end lead flow](./02-end-to-end-flow.md) | How a public record becomes a lead in Slack: pipeline stages, cadence sequence, lead lifecycle |
| 3 | [Collectors](./03-collectors.md) | Each source, how it is parsed and filtered, and its health |
| 4 | [Runtime topology](./04-runtime-topology.md) | Hosts, containers, networks, volumes and ports, today and after the WSL cutover |
| 5 | [Tech stack](./05-tech-stack.md) | Languages, libraries, services and external APIs, by layer |
| 6 | [Data model](./06-data-model.md) | Postgres tables, keys, and how dedup and delivery use them |
| 7 | [Infrastructure as code](./07-iac.md) | IaC layers, phase status, secrets flow, CI/CD and deploy |
| 8 | [Observability](./08-observability.md) | Metrics, logs, alerts and the cadence heartbeat |

## Sources

- Backend code: `/mnt/c/Users/alvin/GolandProjects/groupscout` (`cmd/server`, `internal/`, `migrations/`, `docker-compose*.yml`, `deploy/`)
- Frontend code: `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`
- Prose docs these diagrams complement: [backend/docs/ARCHITECTURE.md](../../backend/docs/ARCHITECTURE.md), [backend/docs/DATA_FLOW.md](../../backend/docs/DATA_FLOW.md), [INFRASTRUCTURE_AS_CODE.md](../../backend/docs/planning/INFRASTRUCTURE_AS_CODE.md), [MONITORING.md](../../backend/docs/guides/MONITORING.md), [CI.md](../../backend/docs/guides/CI.md), [SECRETS.md](../../backend/docs/guides/SECRETS.md), [WSL_HOST.md](../../backend/docs/guides/WSL_HOST.md)

## Known gaps on 2026-10-16

- The Anthropic API credit balance is exhausted, so every Claude enrichment fails (`groupscout-site-n47`). Failed items are retried on the next run. Backend PR #19 adds provider failover; with an OpenAI key set, OpenAI scores leads until Claude is topped up ([AI_PROVIDERS.md](../../backend/docs/guides/AI_PROVIDERS.md)).
- Enrichment failures are reported as "no new leads" (`groupscout-site-chd`).
- Delta permits read a frozen March PDF (`groupscout-site-pkk`).
- `alertd` is not running (`groupscout-site-q74`). The operator UI is built but not deployed, and its `/api/*` routes are not implemented yet (`groupscout-site-eqm`).
