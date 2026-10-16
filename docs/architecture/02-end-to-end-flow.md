# 2. End-to-End Lead Flow

_As of 2026-10-16, backend image `9c69038`._

## 2.1 Pipeline stages

One `/run` walks every collector in order. Each collected item passes through the stages below; the code is `internal/enrichment/enricher.go` and `cmd/server/pipeline.go`.

```mermaid
flowchart TD
    trigger(["n8n cadence or manual POST /run"]) --> lock{"delivery lock free?<br/>delivery_locks, 15 min"}
    lock -->|no| locked(["status: locked"])
    lock -->|yes| idem{"idempotency key<br/>already sent?"}
    idem -->|yes| dup(["status: duplicate"])
    idem -->|no| collect

    subgraph enrich["Enricher.Run: once per collector, once per item"]
        collect["Collector.Collect<br/>scrape / RSS / PDF to RawProject"] --> norm["normalize<br/>default RawData, RawType, Hash"]
        norm --> seen{"hash in<br/>raw_projects?"}
        seen -->|yes| drop(["skip: seen before"])
        seen -->|no| audit["store payload in raw_inputs<br/>(deduped by payload hash)"]
        audit --> ollamaX["Ollama extraction<br/>(off: OLLAMA_EXTRACTION_ENABLED)"]
        ollamaX --> prescore{"rule pre-score<br/>at least ENRICHMENT_THRESHOLD (1)?"}
        prescore -->|no| skipped["insert lead<br/>status = skipped"]
        prescore -->|yes| ai["AI enrichment<br/>Claude Haiku 4.5 (or Gemini)"]
        ai -->|error| retry(["log error, hash NOT recorded<br/>retried next run"])
        ai -->|ok| titledup{"same source + title<br/>already a lead?"}
        titledup -->|yes| dropdup(["skip: duplicate lead"])
        titledup -->|no| ollamaR["Ollama rationale<br/>(off: OLLAMA_SCORING_ENABLED)"]
        ollamaR --> newlead["insert lead<br/>status = new"]
        skipped --> mark["record hash in raw_projects"]
        newlead --> mark
        dropdup --> mark
    end

    mark --> deliver
    subgraph deliver["deliverGuaranteedLeads"]
        cands["candidates: status new or notified,<br/>never sent, priority desc, up to 100"] --> any{"any?"}
        any -->|no| nolead["lead_deliveries: no_eligible_lead"]
        any -->|yes| slackSend["Slack webhook: lead digest"]
        slackSend --> notified["status new to notified"]
        notified --> email["Resend email copy<br/>(best effort)"]
        email --> record["lead_deliveries: one sent row per lead<br/>plus one run row"]
    end
    nolead --> resp(["JSON result + ready-to-post message"])
    record --> resp
```

**Notes**

- The pre-scorer (`internal/enrichment/scorer.go`) is a cheap gate in front of Claude. It scores title, location and description: +2 for Richmond/YVR and nearby places, +1/+2 for value over $1M/$10M, +1 per crew or event keyword, -5 for residential titles or stories outside B.C.
- Skipped items keep their hash, so a later scorer change does not revisit them (`groupscout-site-95y`).
- Delivery sends every eligible lead regardless of Claude's priority score (`groupscout-site-ea0`).
- `new_leads` in the result counts skipped leads too. Enrichment failures are not reported in the result (`groupscout-site-chd`).

## 2.2 Cadence run, step by step

```mermaid
sequenceDiagram
    autonumber
    participant N as n8n<br/>lead-cadence workflow
    participant A as groupscout_app<br/>POST /run
    participant DB as Postgres
    participant S as Sources
    participant C as Anthropic API
    participant SL as Slack
    participant R as Resend
    participant H as Healthchecks.io

    Note over N: Schedule trigger Sun/Tue/Thu 18:00 America/Vancouver
    N->>N: Build cadence key lead-cadence:YYYY-MM-DD:weekday
    N->>N: Already delivered this key? (workflow static data)
    N->>A: GET /health
    A-->>N: {"status":"ok","database":"ok"}
    N->>A: POST /run {guarantee_one_lead, delivery_mode: all_eligible, cadence_key}
    A->>DB: TryAcquireRunLock("lead-delivery")
    A->>DB: lead_deliveries by idempotency key
    loop each collector (8)
        A->>S: fetch page / feed / PDF
        S-->>A: items
        loop each new item that passes the pre-scorer
            A->>C: enrich (title, description, value)
            C-->>A: crew size, duration, priority 1-10, timing, notes
            A->>DB: insert lead, record hash
        end
    end
    A->>DB: ListDeliveryCandidates(100)
    alt candidates found
        A->>SL: lead digest
        A->>DB: status = notified
        A->>R: email copy to LEAD_NOTIFY_EMAILS
        A->>DB: lead_deliveries rows (sent)
    else none
        A->>DB: lead_deliveries (no_eligible_lead)
    end
    A->>DB: release lock
    A-)H: ping success or fail (when CADENCE_HEARTBEAT_URL is set)
    A-->>N: {status, new_leads, notified_leads, delivery_status, message}
    alt delivered
        N->>N: Mark Delivered (static data)
    else no lead, or run failed
        N->>SL: ops notice (OPS_SLACK_WEBHOOK_URL, else the lead channel)
    end
```

## 2.3 Lead lifecycle

```mermaid
stateDiagram-v2
    [*] --> Collected: collector returns RawProject
    Collected --> Dropped: hash already in raw_projects
    Collected --> Skipped: pre-score below threshold
    Collected --> Retry: AI enrichment error
    Retry --> Collected: next run
    Collected --> Dropped: same source + title already a lead
    Collected --> New: enriched and stored
    New --> Notified: cadence sent it (lead_deliveries sent)
    Skipped --> [*]
    Dropped --> [*]
    Notified --> [*]

    note right of New
        UI workflow columns exist (owner, snoozed_until,
        flagged, verification_state) but nothing sets them
        yet; the /api/* routes are planned (groupscout-site-eqm).
    end note
```

Production counts on 2026-10-16: 128 `skipped`, 19 `notified`, 0 `new`.
