# 1. System Context

_As of 2026-10-16._

GroupScout watches public records around Richmond, BC (building permits, tenders, film productions, convention-centre events and news) for projects that will bring crews or delegates to town, and turns them into hotel group-sales leads.

```mermaid
flowchart LR
    sales(["Hotel group-sales team<br/>(Richmond / YVR)"])
    owner(["Owner / operator"])

    subgraph sources["Public sources"]
        direction TB
        richmond["City of Richmond<br/>weekly permit PDFs"]
        delta["City of Delta<br/>permit PDF"]
        creativebc["Creative BC<br/>in-production list"]
        vcc["Vancouver Convention<br/>Centre events"]
        civicinfo["CivicInfo BC<br/>tender RSS"]
        gnews["Google News RSS"]
        announce["BCIB, TransLink,<br/>YVR newsrooms"]
        eventbrite["Eventbrite<br/>Vancouver listings"]
    end

    subgraph gs["GroupScout (home Podman stack)"]
        direction TB
        n8n["n8n<br/>cadence scheduler"]
        api["Go API + pipeline<br/>groupscout_app"]
        pg[("Postgres 17<br/>+ pgvector")]
        obs["Prometheus, Loki,<br/>Alloy, Grafana"]
        alertd["alertd<br/>disruption alerts<br/>(not running)"]
        ui["Operator UI<br/>(built, not deployed)"]
    end

    subgraph ext["External services"]
        direction TB
        claude["Anthropic API<br/>Claude Haiku 4.5"]
        slack["Slack<br/>incoming webhooks"]
        resend["Resend<br/>email"]
        hc["Healthchecks.io<br/>(Phase 9, not configured)"]
        sentry["Sentry"]
        weather["ECCC alerts, YVR flights,<br/>NAV CANADA NOTAMs"]
    end

    sources -->|"HTTP scrape / RSS / PDF"| api
    n8n -->|"POST /run Sun/Tue/Thu 18:00 PT"| api
    api --> pg
    api -->|"enrich + score"| claude
    api -->|"lead digest"| slack
    api -->|"lead email copy"| resend
    api -.->|"heartbeat ping"| hc
    api -.->|"errors"| sentry
    obs -->|"scrape /metrics, ship logs"| api
    obs -.->|"alert rules"| slack
    n8n -->|"no-lead / failure notices"| slack
    weather -.-> alertd
    alertd -.->|"airport disruption alerts"| slack
    ui -.->|"/api/* (planned)"| api

    slack --> sales
    resend --> sales
    owner -->|"Grafana, n8n, psql, logs"| gs
```

**Reading it**

- Solid arrows are live paths. Dotted arrows are built but disabled, or planned.
- The only trigger in production is the n8n schedule. `/run` can also be called by hand with the API token. `/ingest` and `/n8n/webhook` accept single projects pushed from outside.
- Leads leave the system through Slack, with an email copy through Resend. Nothing is exposed outside the PC yet: ingress (Cloudflare Tunnel or Tailscale) is open decision D5 (`groupscout-site-6ah.17`).
