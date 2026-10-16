# 5. Tech Stack

_As of 2026-10-16. Versions are the pinned ones in `go.mod`, `docker-compose.yml`, the Dockerfiles and the CI workflow._

```mermaid
flowchart TB
    subgraph users["Delivery channels"]
        slack["Slack incoming webhooks"]
        resend["Resend email API"]
        ui["Operator UI: Node 22, no npm dependencies,<br/>node:test, static build + same-origin /api proxy<br/>(not deployed)"]
    end

    subgraph orchestration["Orchestration"]
        n8n["n8n 2.43.5<br/>cadence workflow as code"]
    end

    subgraph app["Backend: Go 1.26 (module github.com/alvindcastro/groupscout)"]
        direction LR
        http["net/http API<br/>/run /health /metrics /ingest<br/>/n8n/webhook /digest /leads/{id}/raw"]
        collectors["Collectors<br/>goquery, golang.org/x/net/html,<br/>gofeed, encoding/xml,<br/>poppler pdftotext 25.12"]
        enrich["Enrichment<br/>rule pre-scorer, Claude client,<br/>Gemini client, Ollama client"]
        storage["Storage<br/>pgx v5, modernc sqlite (dev),<br/>golang-migrate, pgvector-go"]
        ops["Ops<br/>prometheus/client_golang, slog,<br/>sentry-go, Healthchecks pinger"]
        alertd["alertd (separate binary)<br/>ECCC, YVR, NAV CANADA,<br/>Slack bot"]
    end

    subgraph ai["AI providers"]
        claude["Anthropic API<br/>claude-haiku-4-5 (default)"]
        gemini["Google Gemini 2.0 Flash<br/>(AI_PROVIDER=gemini, no key set)"]
        ollama["Ollama 0.40.4, mistral<br/>(llm profile, disabled)"]
    end

    subgraph data["Data"]
        pg[("PostgreSQL 17 + pgvector 0.8.7")]
    end

    subgraph obs["Observability (obs profile)"]
        prom["Prometheus v3.15.0"]
        graf["Grafana 13.2.4<br/>provisioned dashboards + alert rules"]
        loki["Loki 3.7.8"]
        alloy["Grafana Alloy v1.20.1"]
        hc["Healthchecks.io (Phase 9)"]
        sentry["Sentry"]
    end

    subgraph platform["Platform and delivery"]
        podman["Podman 5.x + Compose v2<br/>Alpine runtime image"]
        wsl["WSL 2 (Podman machine;<br/>groupscout-prod Ubuntu 26.04)"]
        gha["GitHub Actions CI<br/>GHCR image registry"]
        iac["SOPS 3.13 + age 1.3, gitleaks 8.30,<br/>Ansible 2.21, Renovate,<br/>OpenTofu (planned)"]
        beads["Beads (bd) issue tracker, Dolt-backed"]
    end

    n8n --> http
    http --> collectors --> enrich --> storage --> pg
    enrich --> claude
    enrich -.-> gemini
    enrich -.-> ollama
    http --> slack
    http --> resend
    ui -.-> http
    ops --> prom
    alloy --> loki
    graf --> prom
    graf --> loki
    ops -.-> hc
    ops -.-> sentry
    podman --> wsl
    gha --> podman
```

| Layer | Choice | Why it is there |
|---|---|---|
| Language | Go 1.26, one module, two binaries (`groupscout`, `alertd`) | Single static binary, small Alpine image |
| HTML / feeds | goquery, `x/net/html`, gofeed, `encoding/xml` | Every source is a public page, feed or PDF; no source has an API |
| PDFs | Poppler `pdftotext` (`-bbox-layout` for Richmond, `-layout` for Delta) | Correct font handling; word coordinates for table parsing |
| AI | Claude Haiku 4.5 via the Messages API (512 max tokens) | Cheap structured enrichment; the rule pre-scorer limits calls |
| Database | Postgres 17 + pgvector; SQLite for local dev | Dedup, audit trail, leads, delivery records, embeddings |
| Scheduling | n8n workflow imported from git at start | Retries and ops notices without new code (D11) |
| Notifications | Slack webhooks, Resend email | Where the sales team already works |
| Observability | Prometheus, Loki, Alloy, Grafana, Healthchecks.io, Sentry | Metrics, logs, alert rules, external dead man's switch |
| Runtime | Rootless Podman + Compose on WSL 2 | $0 home hosting; same Ansible moves to a VPS (D1, D4) |
| IaC | SOPS + age, Ansible, GitHub Actions, GHCR, Renovate; OpenTofu planned | See [07-iac.md](./07-iac.md) |
| Frontend | Dependency-free Node 22 UI with `node:test` | Same-origin operator workspace; waits on backend `/api/*` |
