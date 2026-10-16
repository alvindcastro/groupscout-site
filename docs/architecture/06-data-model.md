# 6. Data Model

_As of 2026-10-16: Postgres schema after migrations 001–011, read from the live database. Migration 012 (backend PR #19, not deployed) adds `leads.ai_provider` and `leads.ai_model`._

```mermaid
erDiagram
    raw_projects ||--o{ leads : "raw_project_id (legacy)"
    raw_inputs ||--o{ leads : "raw_input_id"
    leads ||--o{ lead_deliveries : "lead_id"
    leads ||--o{ outreach_log : "lead_id"
    leads ||--o| lead_embeddings : "lead_id"

    raw_projects {
        uuid id PK
        text source
        text external_id
        bytea raw_data
        text raw_type
        text hash UK "dedup key per item"
        timestamptz collected_at
    }
    raw_inputs {
        uuid id PK
        text hash UK "sha256 of payload"
        text payload_type
        bytea payload "full page, feed or PDF"
        text source_url
        text collector_name
        timestamptz created_at
    }
    leads {
        uuid id PK
        uuid raw_project_id FK
        uuid raw_input_id FK
        text source
        text title "dedup with source"
        text location
        bigint project_value
        text general_contractor
        text applicant
        text contractor
        text source_url
        text project_type
        int estimated_crew_size
        int estimated_duration_months
        bool out_of_town_crew_likely
        int priority_score "1-10 from the AI provider, or pre-score"
        text priority_reason
        text rationale
        text suggested_outreach_timing
        text notes
        text status "new | notified | skipped"
        text ai_provider "who scored it (migration 012)"
        text ai_model
        text owner "UI, unused"
        timestamptz snoozed_until "UI, unused"
        bool flagged "UI, unused"
        text verification_state "UI, unused"
        timestamptz created_at
        timestamptz updated_at
    }
    lead_deliveries {
        uuid id PK
        text idempotency_key UK "run key, or run key:lead id"
        text schedule_key "lead-cadence:date:weekday"
        uuid lead_id FK "null for the run row"
        text channel "slack"
        text status "sent | no_eligible_lead | failed"
        text result
        timestamptz sent_at
    }
    delivery_locks {
        text name PK "lead-delivery"
        text owner
        timestamptz acquired_at
        timestamptz expires_at "15 min"
    }
    outreach_log {
        uuid id PK
        uuid lead_id FK
        text contact
        text channel
        text notes
        text outcome
        timestamptz logged_at
    }
    lead_embeddings {
        uuid lead_id PK,FK
        text model
        vector embedding
        timestamptz created_at
    }
    pipeline_runs {
        uuid id PK
        text status
        jsonb sources
        jsonb counts
        jsonb errors
        jsonb request
        timestamptz started_at
        timestamptz finished_at
    }
```

## How the tables are used

```mermaid
flowchart LR
    item["collected item"] -->|"hash exists?"| RP[(raw_projects)]
    item -->|"payload, deduped by sha256"| RI[(raw_inputs)]
    item -->|"lead row; source + title must be new"| L[(leads)]
    L -->|"candidates: new or notified, never sent"| DL[(lead_deliveries)]
    LK[(delivery_locks)] -.->|"one cadence at a time"| DL
    L -.->|"/leads/{id}/raw returns the payload"| RI
```

- `raw_projects.hash` is the "seen before" check. Since PR #18 it is written only after the item reaches a terminal state, so failed enrichments are retried.
- `raw_inputs` is the audit trail: one row per distinct payload. Feed and page collectors store the whole document, so many leads share one row.
- `lead_deliveries` makes cadence runs idempotent: one row per run key and one per delivered lead. A lead with a `sent` row is never offered again.

Row counts on 2026-10-16: `raw_projects` 169, `raw_inputs` 65, `leads` 147 (128 skipped, 19 notified), `lead_deliveries` 44; `outreach_log`, `lead_embeddings` and `pipeline_runs` are empty.
