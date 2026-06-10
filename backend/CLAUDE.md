# CLAUDE.md — groupscout

> Context file for Claude Code sessions in GoLand.
> Project: https://github.com/alvindcastro/groupscout

---

## Who I Am

- **Alvin** — Programmer Analyst at Douglas College. Wife is Hotel Sales Manager at Sandman Hotel Vancouver Airport (Richmond, BC) — he built this tool for her team.
- Strong Go background, JetBrains GoLand, Bruno for API testing
- Prefer phased, documented builds. Run `bd ready` / `bd show <id>` for live work, then use `docs/planning/PHASES.md` as phase context before writing code.

---

## Project Goal

**groupscout** is a group lodging demand intelligence tool for hotel sales teams.

It monitors public data sources for signals that indicate incoming groups needing room blocks — starting with construction crews in Richmond, BC, then expanding to sports teams, film productions, government contractors, and touring acts.

Output: prioritized lead intelligence delivered through Slack/email plus the separate operator UI. Backend/API work belongs in `/mnt/c/Users/alvin/GolandProjects/groupscout`; UI work belongs in `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

---

## Context — Sandman Hotel Vancouver Airport

- Located in Richmond, BC (near YVR)
- Target segment to start: construction crew lodging
- Key selling points: YVR proximity, parking for work trucks, Shark Club F&B, extended stay rates, direct billing
- Sales team receives output via Slack weekly digest

---

## Stack

| Layer | Technology | Status |
|---|---|---|
| Language | Go 1.26 | ✅ |
| Database | Postgres (`pgx`) / SQLite (`modernc.org/sqlite`) | ✅ |
| Vector DB | `pgvector` (Postgres) / Go-native (SQLite) | ✅ |
| AI enrichment | Claude Messages API / Gemini / Ollama (local LLMs) | ✅ |
| Notifications | Slack incoming webhook (Block Kit) | ✅ |
| Config | Environment variables + `.env` loader (12-factor) | ✅ |
| Scraping | `github.com/PuerkitoBio/goquery` | ✅ |
| RSS parsing | `github.com/mmcdole/gofeed` | ✅ |
| Scheduler | n8n / HTTP trigger | ✅ |
| Migrations | `golang-migrate/migrate` | ✅ |
| Email | Resend | ✅ |
| Monitoring | Sentry | ✅ |
| Deployment | Docker, VPS or Railway | ✅ |

---

## Architecture

```
Scheduler (cron)
    │
    ▼
Collector Layer (Mini Phase A)   ← scrapers + parsers
    │ raw project data
    ▼
Enrichment Layer (Mini Phase B)  ← Claude API + rule scorer
    │ structured lead
    ▼
Storage Layer (Mini Phase C)     ← SQLite / Postgres
    │ new/updated leads
    ▼
Notification Layer (Mini Phase D) ← Slack digest + email
```

---

## Project Structure

```
groupscout/
├── api/
│   └── swagger.yaml           # OpenAPI specification
├── cmd/
│   ├── server/
│   │   └── main.go            # --run-once pipeline entry point + HTTP trigger
│   ├── alertd/
│   │   └── main.go            # airport disruption alert daemon (Phase 17) ✅
│   └── tools/
│       ├── smoketest/
│       │   └── main.go        # dev tool: live collector run / -rawpdf / -testslack
│       └── test_sentry/
│           └── main.go        # verify Sentry connectivity
├── internal/
│   ├── collector/
│   │   ├── permits/           # Richmond, Delta
│   │   ├── events/            # VCC, Eventbrite, CreativeBC
│   │   ├── news/              # News RSS, BC Bid, Announcements
│   │   ├── utilities/         # BCHydro, FortisBC (Phase 13.1) 🔄
│   │   └── collector.go       # Collector interface + RawProject struct
│   ├── enrichment/
│   │   ├── claude.go          # Claude Messages API client ✅
│   │   ├── claude_test.go     # unit tests: stripMarkdown, extractText ✅
│   │   ├── enricher.go        # orchestrates collect→dedup→enrich→store ✅
│   │   ├── enricher_test.go   # unit tests: toLeadRecord ✅
│   │   └── scorer.go          # rule-based pre-scorer (Phase 3-B)
│   ├── storage/
│   │   ├── audit.go           # AuditStore interface + Postgres impl ✅
│   │   ├── db.go              # Open() + Migrate() (idempotent schema)
│   │   ├── raw.go             # RawProjectStore interface + SQLite impl
│   │   ├── leads.go           # LeadStore interface + Lead struct + SQLite impl
│   │   └── embeddings.go      # EmbeddingStore interface + pgvector/SQLite impl ✅
│   ├── leadnotify/
│   │   ├── slack.go           # Slack webhook for lead notifications ✅
│   │   ├── slack_test.go      # lead notification tests ✅
│   │   └── email.go           # Resend email digest (Phase 4-C)
│   ├── scheduler/
│   │   └── scheduler.go       # cron wiring (Phase 3-D)
│   ├── weather/
│   │   └── eccc.go            # ECCC GeoJSON API poller (Phase 17) ✅
│   ├── aviation/
│   │   ├── yvr.go             # YVR flight scraper (Phase 17) ✅
│   │   ├── navcanada.go       # NOTAM parser (Phase 17) ✅
│   │   └── scorer.go          # SPS calculation (Phase 17) ✅
│   ├── ollama/
│   │   ├── client.go          # Ollama HTTP client ✅
│   │   ├── extractor.go       # NLP signal extraction ✅
│   │   ├── scorer.go          # lead scoring rationale ✅
│   │   ├── alertcopy.go       # disruption alert copy generation ✅
│   │   ├── metrics.go         # Prometheus metrics ✅
│   │   └── modelfile_manager.go # Modelfile hot-swap ✅
│   └── alert/
│       ├── lifecycle.go       # disruption alert state machine (Phase 17) ✅
│       └── slack.go           # Slack chat.update notifier (Phase 17) ✅
├── migrations/
│   ├── 001_init.up.sql
│   └── 001_init.down.sql
├── config/
│   └── config.go
├── CLAUDE.md
├── docs/
│   ├── planning/
│   │   ├── PHASES.md          # phase reference and historical build context
│   │   ├── ROADMAP.md
│   │   ├── OLLAMA_INTEGRATION.md
│   │   └── groupscout-plan.md
│   ├── guides/
│   │   ├── SETUP.md
│   │   ├── DOCKER.md
│   │   ├── TESTING.md
│   │   ├── OLLAMA_SETUP.md
│   │   ├── ALERTD_SETUP.md
│   │   ├── TROUBLESHOOTING.md
│   │   ├── N8N_GUIDE.md         # § 11 Slack Integration; § 10 Workflow Inventory
│   │   ├── HOME_DEPLOY.md
│   │   └── historical LUX guides migrated out of this docs tree
│   ├── mvps/
│   │   ├── mvp-a.md             # MVP-A spec: AI client status email generator
│   │   ├── mvp-b.md             # MVP-B spec: automated lead follow-up sequence
│   │   ├── mvp-c.md             # MVP-C spec: LinkedIn & podcast post pipeline
│   │   ├── mvp-a/               # payloads, prompts, n8n_workflow.json
│   │   ├── mvp-b/               # payloads, prompts, n8n_workflow.json
│   │   └── mvp-c/               # payloads, prompts, n8n_workflow.json
│   ├── prompts/               # phase-specific AI coding prompts
│   ├── superpowers/
│   │   ├── specs/             # design docs per phase
│   │   └── plans/             # implementation plans per phase
│   ├── CHANGELOG.md           # plain-English knowledge base
│   ├── ARCHITECTURE.md
│   ├── API_CONFIG.md
│   └── API_TESTING.md
├── go.mod
├── go.sum
├── AGENTS.md
└── README.md
```

---

## Key Interfaces

```go
// All data sources implement this
type Collector interface {
    Name() string
    Collect(ctx context.Context) ([]RawProject, error)
}
```

```go
// RawProject is the data contract between collectors and the enrichment pipeline.
// Metadata carries structured per-source fields (contact info, source_type, closing_date, etc.)
type RawProject struct {
    Source      string
    ExternalID  string
    Title       string
    Description string
    Location    string
    Value       int64
    IssuedAt    time.Time
    Hash        string
    RawData     []byte         // raw payload for audit trail (Phase 27)
    RawType     string         // "pdf", "rss", "html", "json"
    Metadata    map[string]any // structured pipeline data; keys vary by collector
}
```

---

## Design Rules

- **Repository pattern** — all DB access behind interfaces, storage is swappable
- **Collector interface** — adding a new source = new struct, core pipeline unchanged
- **Claude for enrichment only** — scraping stays deterministic Go; AI only for interpretation
- **Rule-based pre-scorer** — filter noise in Go before spending Claude API tokens
- **Slack/email remain notification channels** — the operator UI is developed separately from backend pipeline code
- **Outreach stays manual** — Claude drafts the email, human sends it. No auto-send.
- **12-factor config** — all secrets and config via env vars
- **SQLite first** — zero-ops for local dev; swap to Postgres for deployment
- **UI is a sibling implementation track** — coordinate backend `/api/*` contracts here, but implement UI code in the frontend repo

---

## Build Progress

Use `bd` for live task state. `docs/planning/PHASES.md` is retained as the phase-reference checklist and historical build context.

### GroupScout Core Pipeline

| Phase | Goal | Status |
|---|---|---|
| Phase 1 | Foundation — DB boots, schema applied | ✅ complete |
| Phase 2 | Richmond → Claude → Slack (one full pipeline) | ✅ complete |
| Phase 3 | Dedup hardened, BC Bid/Delta added, n8n trigger | ✅ complete |
| Phase 4 | Creative BC/VCC sources live, more to follow | ✅ complete |
| Phase 5 | Smart refresh (avoid redundant PDF fetches) | deferred |
| Phase 6 | Productionize — Docker, Postgres, VPS deploy | ✅ complete |
| Phase 7 | User requests & UX refinements | 🔄 in progress |
| Phase 8 | Observability — structured logging, Sentry, health checks | 🔄 in progress |
| Phase 14 | Infrastructure — Docker Compose, n8n, Prometheus, Loki | 🔄 in progress |
| Phase 15 | PostgreSQL + pgvector migration | ✅ complete |
| Phase 16 | LLM provider abstraction (Claude / Gemini / Ollama) | 📋 planned; current Ollama runtime is separate |
| Phase 17 | Airport disruption alert system (`cmd/alertd/`) | ✅ complete |
| Phase 20 | Housekeeping & developer experience | ◐ baseline complete; broader smell refactors open in `groupscout-site-2h1` |
| Phase 21 | Ollama prod hardening | ✅ complete |
| Phase 13.1 | BC Hydro & FortisBC utility collectors | 🔄 in progress |
| Phase 27 | Input audit & verification trail | ✅ raw audit/redaction/retention complete; sanitized preview open in `groupscout-site-4cv` |
| Phase 22 | Multi-property support — config-driven portfolio routing | 📋 planned |
| Phase 23 | Advanced intelligence — repeat detection + signal quality scoring | 📋 planned |

### LUX Platform Pipelines (n8n, Claude API — no Go code)

Standalone n8n workflows for LUX Construction. Run independently — only n8n needed, no GroupScout server required. The old `docs/mvps/` and `docs/guides/LUX_MVP_*` files are historical/migrated in this coordinator repo; use the available LUX Codex skills in `backend/plugins/groupscout-agents/skills/` and the backend source repo's `.claude/agents/` specs as the current agent-role references.

| MVP | Goal | Status |
|---|---|---|
| MVP-A | AI client status email generator (JobTread → Claude → Slack review) | ✅ complete |
| MVP-B | Automated lead follow-up sequence (classify → route → 3-email → Airtable → Slack) | ✅ complete |
| MVP-C | LinkedIn & podcast post pipeline (milestone/episode → Claude → Slack review) | ✅ complete |

**Agent roles / skills:** `.claude/agents/lux-status-email-writer.md`, `lux-lead-classifier.md`, `lux-lead-sequence-writer.md`, `lux-content-writer.md`, `lux-slack-notifier.md`; Codex mirrors live under `backend/plugins/groupscout-agents/skills/`.

---

## Env Vars

```env
# Required for pipeline
DATABASE_URL=groupscout.db          # SQLite file path (local) or postgres:// (production)
CLAUDE_API_KEY=sk-ant-...
SLACK_WEBHOOK_URL=https://hooks.slack.com/...

# Pipeline tuning
MIN_PERMIT_VALUE_CAD=500000         # filter threshold; lower to 100000 during testing
ENRICHMENT_ENABLED=true
PRIORITY_ALERT_THRESHOLD=9

# Scheduler (Phase 3-D) — run Wed + Sun to catch Richmond's weekly publish
DIGEST_DAY=wednesday                # day of digest run
DIGEST_HOUR=8                       # 8am local time

# Ollama (Local LLM)
OLLAMA_ENABLED=true
OLLAMA_ENDPOINT=http://localhost:11434
OLLAMA_MODEL=mistral

# Future sources (Phase 3+)
# RESEND_API_KEY=re_...
# BCBID_RSS_URL=
# NEWS_API_KEY=
```

---

## Claude Enrichment — Expected JSON Output

```json
{
  "general_contractor": "string or unknown",
  "project_type": "civil|commercial|industrial|utility|residential|unknown",
  "estimated_crew_size": 80,
  "estimated_duration_months": 6,
  "out_of_town_crew_likely": true,
  "priority_score": 9,
  "priority_reason": "Large civil project within 5km of hotel, groundbreaking imminent",
  "suggested_outreach_timing": "Reach out now — crews mobilizing in 4-6 weeks",
  "notes": "GC is PCL Construction. Check LinkedIn for travel coordinator."
}
```

---

## DB Schema (abbreviated)

```sql
CREATE TABLE raw_projects (
  id           UUID PRIMARY KEY,
  source       TEXT NOT NULL,
  external_id  TEXT,
  raw_data     JSONB NOT NULL,
  collected_at TIMESTAMPTZ NOT NULL,
  hash         TEXT UNIQUE NOT NULL
);

CREATE TABLE leads (
  id                        UUID PRIMARY KEY,
  raw_project_id            UUID REFERENCES raw_projects(id),
  source                    TEXT,
  title                     TEXT,
  location                  TEXT,
  project_value             BIGINT,
  general_contractor        TEXT,   -- Claude's best inference of the GC
  applicant                 TEXT,   -- raw applicant from permit (may include phone)
  contractor                TEXT,   -- raw contractor from permit (may include phone)
  project_type              TEXT,
  estimated_crew_size       INT,
  estimated_duration_months INT,
  out_of_town_crew_likely   BOOLEAN,
  priority_score            INT,
  priority_reason           TEXT,
  suggested_outreach_timing TEXT,
  notes                     TEXT,
  status                    TEXT DEFAULT 'new',
  created_at                TIMESTAMPTZ NOT NULL,
  updated_at                TIMESTAMPTZ NOT NULL
);

CREATE TABLE outreach_log (
  id        UUID PRIMARY KEY,
  lead_id   UUID REFERENCES leads(id),
  contact   TEXT,
  channel   TEXT,
  notes     TEXT,
  outcome   TEXT,
  logged_at TIMESTAMPTZ NOT NULL
);
```

---

## Future Segments (Phase 4+)

When construction is working well, expand collectors to:
- Sports teams (Rogers Arena schedule, BC Lions, Vancouver FC)
- Film/TV production (BC film permits)
- Government contractors (DND, CBSA, Transport Canada near YVR)
- Touring acts and conferences

The Collector interface makes this additive — no core changes needed.

---

*groupscout — group lodging demand intelligence*
*Sandman Hotel Vancouver Airport, Richmond BC*

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->
