# Future Integration Roadmap

> Maps brainstorm ideas to concrete GroupScout phases and PHASES.md tasks.
> Last updated: 2026-05-25

> **Roadmap ownership:** [PHASES.md](./PHASES.md) is the canonical atomic tracker and [ROADMAP.md](./ROADMAP.md) is the strategic summary. This file is an idea map; keep phase numbers aligned with those canonical docs when editing it.

---

## 1. Agentic Engineering & GenAI Workflows

*Target: `internal/enrichment`, `internal/tools`*
*Planned phase: **Phase 24 — Agentic Reasoning & Tool-Calling***

| Idea | Status | Phase |
|---|---|---|
| Reasoning Loops (ReAct/Plan-and-Solve) for complex permits | 📋 planned | Phase 24-A |
| RAG: top-k similar past leads injected into prompt context | ✅ done (pgvector, Phase 15) | — |
| Tool-Calling: BC Registry lookup + LinkedIn URL generation | 📋 planned | Phase 24-B |
| Multimodal PDF parsing via Claude vision | ⏸ deferred (5+ cities threshold) | Phase 24-C |
| Multi-agent pipeline: one agent per source, coordinator deduplicates | 📋 future | Phase 26+ |

**Next concrete task:** `internal/enrichment/react_test.go` — write `TestReactEnricher_CallsToolOnBorderlineScore` with mock LLM; fail first. Prerequisite provider abstraction is tracked in `groupscout-site-vud`.

---

## 2. Data Foundation & AI Pipelines

*Target: `internal/collector`, `internal/storage`, `internal/enrichment`*
*Planned phase: **Phase 25 — AI Observability & Quality***

| Idea | Status | Phase |
|---|---|---|
| AI-Ready SQL (`v_lead_context` view + `GetContext()`) | 📋 planned | Phase 25-A |
| Prompt refactor to use `GetContext()` instead of hand-built strings | 📋 planned | Phase 25-A |
| Langfuse LLM observability (token cost, latency, prompt version) | 📋 planned | Phase 25-B |
| Enrichment eval harness (`EvalLead()` structural validation) | 📋 planned | Phase 25-C |
| AI observability eval (RAGAS / Vertex Eval for hallucination detection) | 📋 future | Phase 26+ |
| Unstructured PDF ingestion (AI-driven parsing for tender docs) | 📋 future | Phase 24-C |

**Next concrete task:** `migrations/005_ai_context.up.sql` — `v_lead_context` view; then `TestGetContext_ReturnsExpectedString`. Beads: `groupscout-site-48g`.

---

## 3. Integration & Cloud-Native Development

*Target: `config`, `terraform/`, `cmd/server/`*
*Planned phase: **Phase 26 — Cloud-Native & Event-Driven***

| Idea | Status | Phase |
|---|---|---|
| LLM Provider Abstraction (`LLMClient` interface, factory) | 📋 planned | Phase 16 |
| Hetzner CX32 + Coolify production deploy (primary recommendation, ~$10/month) | 📋 planned | Phase 26-B |
| Event-driven ingestion: `POST /ingest` → `EnrichOne()` (platform-agnostic) | ✅ implemented in backend branch `task/event-driven-ingest` | Phase 26-C |
| Terraform IaC for GCP (optional; alertd incompatible with Cloud Run — needs Compute Engine VM) | 📋 optional | Phase 26-D |
| AIaaS API Layer: expose enrichment as standalone service | 📋 future | Phase 10 |
| CRM/ERP Integration: HubSpot, Salesforce, SAP | 📋 future | Phase 11 |

**Next concrete task (Phase 26-A/B/D):** Home deploy smoke and restore are tracked in `groupscout-site-39g`; Coolify guide/backup setup is documented in `docs/guides/COOLIFY.md`; event-driven `/ingest` was implemented under `groupscout-site-b25`. The remaining Postgres raw-input FK integration failure from the ingest test pass is tracked in `groupscout-site-wda`.

---

## 4. Agile Execution & Quality

*Target: `docs/guides/TESTING.md`, `internal/enrichment/eval.go`*

| Idea | Status | Phase |
|---|---|---|
| TDD-first rule enforced per phase (T tasks before impl tasks) | ✅ documented | All phases 16+ |
| Enrichment structural eval (`EvalLead()` + Sentry on hard failures) | 📋 planned | Phase 25-C |
| UAT suites validating AI-generated endpoints + outreach drafts | 📋 planned | Phase 25-C |
| Benchmark test for pipeline throughput (target: <30s for 20 permits) | 📋 future | Phase 9 |

---

## Summary of Next Steps

This file is an idea map, not the active priority queue. Use [NOT_DONE_AND_UPGRADES.md](./NOT_DONE_AND_UPGRADES.md) and Beads for current sequencing.

Current Beads-backed priority themes:

1. Source-of-truth hygiene and source checkout reconciliation.
2. Runtime correctness for collector/raw persistence warnings.
3. Backend UI API routes and generated frontend client/types.
4. Observability, AI-ready SQL, provider abstraction, analytics, and deployment upgrades.

Idea-level dependencies still matter: the lead-management API work unblocks richer Slack/UI actions, the LLM provider abstraction should precede agentic tool-calling, and AI-ready SQL should inspect current migrations before choosing filenames.
