# 8. Observability

_As of 2026-10-16. Runbook: [MONITORING.md](../../backend/docs/guides/MONITORING.md). Phase 9 is live from PR #15; the Healthchecks.io account and ops Slack channel are owner steps (`groupscout-site-6ah.16`)._

```mermaid
flowchart LR
    subgraph stack["Podman stack (obs profile)"]
        app["groupscout_app<br/>/metrics, /health, slog to stdout"]
        n8n["groupscout_n8n"]
        prom["Prometheus<br/>scrapes groupscout:8080/metrics"]
        alloy["Alloy<br/>discovers containers via Podman socket"]
        loki["Loki"]
        graf["Grafana<br/>provisioned datasources, dashboards,<br/>5 alert rules"]
    end

    subgraph outside["Outside the stack"]
        hc["Healthchecks.io check groupscout-cadence<br/>cron 0 18 * * 0,2,4 Vancouver, 45 min grace"]
        ops["Ops Slack channel<br/>(OPS_SLACK_WEBHOOK_URL, not set yet)"]
        lead["Lead Slack channel"]
        email["Owner email"]
        sentry["Sentry"]
    end

    app -->|"metrics"| prom
    app -->|"container logs"| alloy
    n8n -->|"container logs"| alloy
    alloy --> loki
    prom --> graf
    loki --> graf
    graf -->|"alerts, never the lead channel"| ops
    app -.->|"ping after each cadence run<br/>(CADENCE_HEARTBEAT_URL unset)"| hc
    hc -->|"missed or /fail ping"| email
    app -.->|"exceptions"| sentry
    n8n -->|"no-lead and health-preflight notices<br/>(falls back to the lead channel)"| lead
```

## Signals

| Signal | Where it comes from | Alert |
|---|---|---|
| `up{job="groupscout"}` | Prometheus scrape | API down for 5 min |
| `groupscout_cadence_overdue` | `cmd/server/monitoring.go`, schedule in `internal/cadence` | Cadence missed (in-stack backup of the dead man's switch) |
| `groupscout_collector_runs_total{result="error"}` | `Enricher.OnCollectorRun` | A collector failed (one alert per collector) |
| `groupscout_last_lead_created_timestamp_seconds` | `MAX(created_at) FROM leads`, then each run | No new leads in 14 days |
| Startup log lines for `groupscout_app` / `groupscout_n8n` | Loki | Container restart loop |
| Cadence heartbeat | API pings Healthchecks.io after each cadence run | Missed or failed cadence: n8n, API, Podman or PC down |
| `groupscout_llm_provider_failures_total{reason="account_error"}` | `internal/llm` chain (PR #19) | AI provider account problem: no credit, quota, bad key, unknown model |
| `groupscout_llm_exhausted_total` | `internal/llm` chain (PR #19) | AI enrichment failed on every provider |
| `/health` `ai`, `ai_providers` | `internal/llm` chain status (PR #19) | none; read by hand or by the n8n preflight |

The schedule appears in three places (n8n trigger, `internal/cadence.Default`, `deploy/config/healthchecks/checks.json`), and `internal/smoke/monitoring_test.go` fails if they drift.

## Blind spots found on 2026-10-16

```mermaid
flowchart LR
    claude["Claude returns 400<br/>credit balance too low"] --> enr["Enricher logs error,<br/>skips the item"]
    enr --> result["/run: status success,<br/>no_eligible_lead,<br/>'every lead already delivered'"]
    enr --> skippedRows["skipped leads still inserted"]
    skippedRows --> metric["last_lead_created keeps moving"]
    metric --> quiet["No new leads in 14 days<br/>never fires"]
    result --> n8nmsg["n8n posts 'no new leads'"]
    enr -.->|"not counted"| collectorMetric["collector_runs_total{result=error}<br/>counts collection errors only"]
```

None of the alerts fire when every AI enrichment fails, so the 2026-10-07 to 10-16 credit outage went unnoticed. Backend PR #19 closes the metric and alert part: the provider chain fails over to the next provider, and the two `llm-rules.yml` alerts fire on account errors and on total failure. Still open in `groupscout-site-chd`: report enrichment failures in the run result and message, and count only real (non-skipped) leads for the 14-day rule.
