# AI.md — LLM Integration Roadmap for groupscout

> Brainstorm of every AI/LLM integration worth considering.
> Current use: Claude Haiku for permit enrichment (score, GC inference, crew size, outreach timing).

---

## What We Have Now

Every new permit gets a single Claude Haiku call that returns:
- `general_contractor` — best inference from applicant/contractor names
- `project_type` — civil / commercial / industrial / etc.
- `estimated_crew_size` — rough headcount
- `estimated_duration_months`
- `out_of_town_crew_likely` — boolean
- `priority_score` (1–10) + `priority_reason`
- `suggested_outreach_timing`
- `notes` — actionable intel (check LinkedIn, ask for travel coordinator, etc.)

Cost: ~$0.001/permit with Haiku. For 5 permits/week that's ~$0.005/week — effectively free.

---

## Near-Term (Phase 3–4)

### 1. Outreach Email Draft (Phase 4-D4) ← already in PHASES.md
Claude drafts a cold outreach email per lead. Human reviews and sends manually.

**Prompt approach:** Give Claude the full lead record + hotel context (Sandman, YVR proximity, truck parking, Shark Club). Ask for a 3-sentence intro email addressed to the GC's travel coordinator.

**Why this is high value:** The bottleneck after finding the lead is writing the first email. Draft-ready output turns a 5-minute task into a 30-second review.

---

### 2. Rule-Based Pre-Scorer → AI-Assisted Pre-Scorer
Currently PHASES.md has a pure Go rule-based pre-scorer (Phase 3-B). Consider a hybrid:

- **Go rules first:** Eliminate obvious noise fast (residential, value < threshold) — no token cost.
- **Lightweight Claude pass second:** For borderline permits (score 4–6 from rules), ask Claude a single yes/no question: *"Is this permit likely to involve an out-of-town construction crew?"* — one sentence, 10 tokens. Costs almost nothing, catches edge cases the rules miss.

---

### 3. Cross-Source Deduplication
Richmond and Delta will sometimes permit the same large project (one city issues, the other references). A pure hash dedup won't catch this — the permit numbers and addresses differ.

**Option A — Claude semantic check:** When a new permit comes in, fetch the 5 most recent leads from the DB and ask Claude: *"Is this the same project as any of these?"* — low cost, runs only on new permits.

**Option B — Embedding similarity:** Embed permit title + address + GC name with a small embedding model. Flag pairs above a cosine threshold for human review. More robust for large lead volumes but adds infrastructure.

**Recommendation:** Option A is good enough until lead volume justifies embeddings.

---

### 4. GC Contact Enrichment (web search)
Claude's inference of the GC name is good but contact info (email, LinkedIn) requires a web lookup.

**Claude + web search tool:** When a permit scores ≥ 8, trigger a Claude call with the `web_search` tool enabled. Ask it to find the GC's main office phone, project manager name, and LinkedIn company page. Include the result in the lead's `notes` field.

**Cost:** Only for high-priority leads. At $0.01–0.05/search call, still cheap for 2–3 leads/week.

---

## Medium-Term (Phase 4–5)

### 5. News Article Summarization
Phase 4-A adds a news collector (Google News RSS). Raw news snippets aren't leads — Claude converts them.

**Flow:** Fetch article headline + first 500 chars → Claude: *"Does this article signal an upcoming construction project near Richmond or Delta BC? If yes, extract: project name, location, estimated value, expected start date."*

Filter on Claude's yes/no before storing. Only articles that pass become leads.

---

### 6. BC Bid Contract Parsing (Phase 3-D) ✅
BC Bid notices are semi-structured prose. A regex parser will break constantly.

**Approach implemented:** Hybrid/AI-Ready collector.
- **Automated:** Fetches construction/road award notices via CivicInfo BC RSS feeds.
- **Manual/n8n:** Accepts raw HTML/JSON via `/run` endpoint for complex notices.
- **AI Extraction:** Feeds raw text to Claude for structured field extraction (awardee, value, duration).
- **Automation:** n8n can trigger the pipeline and inject raw data from the main BC Bid portal.

---

### 7. Announcement Summarizer (Phase 4-A)
BCIB, TransLink, YVR newsroom pages have press releases in plain prose. Claude reads them better than regex.

**Note:** Creative BC production list is already implemented (Phase 4-A2) using a similar approach.

**Flow:** Scrape page → extract article text → Claude: *"Does this announcement signal a large construction project near Richmond BC that would require a construction crew? Summarize in 2 sentences and estimate project value."*

---

### 8. Multimodal PDF Parsing (future)
Instead of shelling out to `pdftotext`, pass the PDF directly to Claude with vision enabled.

**Why this is interesting:** Handles any PDF format automatically — no custom parser per city, no `pdftotext` dependency.

**Why to wait:** Costs ~10–50x more per permit than the current approach ($0.01–0.05 vs $0.001). Only worth it if we're adding 5+ new cities and don't want to write a parser for each.

---

## Longer-Term / Experimental

### 9. Conversational Lead Query Interface
A simple CLI: `groupscout ask "show me all industrial leads in the last 30 days over $1M"`

Claude translates the natural language query into a DB query, fetches results, and formats them as a plain-English summary. No SQL knowledge needed for the sales team.

**Implementation:** Claude + function calling. Expose a set of `query_leads(filters)` functions Claude can call.

---

### 10. Extended Thinking for Complex Scoring
For permits where the score is ambiguous (industrial but low value, residential but $20M), use Claude Sonnet with extended thinking enabled. Thinking budget: 2000 tokens. More expensive (~$0.02/call) but only triggered on genuinely unclear cases.

**Trigger condition:** Haiku pre-score is 5–7 AND value > $2M.

---

### 11. Multi-Agent Pipeline (future architecture)
As sources multiply (Richmond, Delta, BC Bid, news, film permits, sports), a single enricher loop gets slow.

**Agent-per-source:** Each collector runs as an independent agent. A coordinator agent merges results, deduplicates cross-source, and ranks the final digest.

**Claude Agent SDK** would handle orchestration. Each source agent runs in parallel, coordinator waits for all.

---

### 12. Digest Personalization
Today: one Slack digest, all leads, same format.

Future: Claude tailors the digest based on what the sales manager cares about this week. *"You have 3 leads. The Vulcan Way warehouse is highest priority — GC flies crews from Alberta. The Ladner industrial is borderline. The restaurant renovation is low priority."*

**Input:** Current leads + past outreach log (which leads were contacted, what happened). Claude writes the narrative.

---

### 13. AI-Ready SQL + RAG (Local-First, No New Infrastructure)

> Full exploration: see [AI_DATA_STRATEGY.md](./AI_DATA_STRATEGY.md).

These two ideas from `FUTURE_INTEGRATION.md` directly enable each other and are the highest-leverage near-term AI upgrade.

**AI-Ready SQL** replaces hand-crafted prompt strings in `claude.go` with a `v_lead_context` SQLite view — one place builds the LLM context, used everywhere.

**RAG** embeds each lead's context string, stores it in `lead_embeddings`, and retrieves the top-3 most similar past leads before every Claude call. Claude then enriches with historical intelligence instead of guessing from scratch.

**They work together:**
```
New permit → AI-Ready SQL builds context_text
           → embed context_text (Voyage AI, free tier)
           → find top-3 similar past leads
           → inject as context into Claude prompt
           → Claude gives better GC inference + crew size + score
```

**No new infrastructure needed today:**
- Similarity search runs in Go (cosine similarity on in-memory float32 slices — fast for <1000 leads)
- Embeddings stored as JSON arrays in SQLite `lead_embeddings` table
- Phase 6 Postgres migration: swap to pgvector; repository pattern means only the storage impl changes

**Added cost:** ~$0/week (Voyage AI free tier covers current volume)

---

## Model & Provider Selection Guide

> Current implementation is hardcoded to Anthropic. For provider abstraction (no lock-in), see [AI_DATA_STRATEGY.md — LLM Provider Abstraction](./AI_DATA_STRATEGY.md).
> Short version: one `LLMClient` interface, two concrete impls (Claude + OpenAI-compatible), config-driven.

### By use case (Claude defaults)

| Use case | Default model | Alternative | Why |
|---|---|---|---|
| Permit enrichment (bulk) | Haiku 4.5 | GPT-4o-mini, Groq Llama 3.1 8B | ~$0.001/call, structured JSON, fast |
| Ambiguous scoring | Sonnet 4.6 | GPT-4o, Mistral Large | Better reasoning |
| Email drafting | Sonnet 4.6 | GPT-4o | Quality matters for outreach |
| Complex scoring (extended thinking) | Sonnet 4.6 + thinking | o1-mini (OpenAI) | Best for borderline high-value permits |
| Web search enrichment | Sonnet 4.6 + tools | GPT-4o + tools | Tool use reliability |
| Conversational query | Haiku 4.5 | GPT-4o-mini, Ollama Llama 3.2 | Low stakes, fast, cheap |
| Multimodal PDF parsing | Sonnet 4.6 | GPT-4o (vision) | Vision capability |
| Dev / local testing | — | Ollama (any model) | Free, no API key, no data sent externally |

### By provider

| Provider | Hosted by | Best fit | Data residency |
|---|---|---|---|
| **Anthropic** | US cloud | Default — best quality for permit enrichment | US only |
| **OpenAI** | US cloud | Drop-in fallback via `OpenAICompatibleClient` | US only |
| **Azure OpenAI** | Microsoft Azure | Enterprise / compliance use case; Canadian region available | Canada possible |
| **Groq** | US cloud | High-volume batch runs (fastest inference) | US only |
| **Mistral AI** | EU cloud | GDPR-sensitive data | EU |
| **Ollama** | Self-hosted (Docker) | Dev, testing, air-gapped environments | Local only |

---

## Cost Estimate at Scale

Assuming 20 permits/week across all sources, 5 pass filter:

| Integration | Cost/week |
|---|---|
| Current enrichment (Haiku × 5) | ~$0.005 |
| + Email drafts (Sonnet × 5) | ~$0.05 |
| + Web search for score ≥ 8 leads (×2) | ~$0.10 |
| + News summarization (×20 articles) | ~$0.02 |
| **Total** | **~$0.18/week → ~$9/year** |

Effectively free at this volume.

---

*groupscout AI roadmap — local planning doc, not committed*
