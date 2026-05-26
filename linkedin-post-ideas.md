# LinkedIn Post Ideas

These ideas come from the centralized project documentation, including the backend README, planning docs, build log, UI strategy, EvalOps plan, and frontend workflow docs. They now also reflect the two sample LinkedIn posts used as voice anchors for `linkedin-post-drafts.md`.

## Voice Anchors

- Start from a concrete build moment, not a broad claim.
- Use "my side project, GroupScout" for the first full reference when the name helps, then use "the system," "the project," or "this side project."
- Use first person when the post is about a lesson learned while building.
- Name specific tools and failure modes: `pdftotext`, malformed JSON, missing PDFs, parser layout drift, duplicate records, Docker runtime packages.
- Let the lesson come after the detail.
- Prefer sober product language: visibility, responsibility, uncertainty, evidence, workflow, ownership, review.
- Treat AI as one part of the system, not the whole system.

## Build Story

- The first end-to-end pipeline fetched a static PDF, extracted text, regexed the first projects, scored them with Claude, and sent JSON to Slack.
- The first working Slack lead was a visibility milestone, not a scale milestone.
- Public data was already available. The missing piece was a workflow that read it every week and turned it into sales action.
- My side project, GroupScout, started with one hotel, one city permit report, and one practical question: which projects will need rooms?
- A small Go service can replace manual weekly scanning across public sources.

## Engineering Lessons

- PDF parsing fails in surprising ways. Content-aware parsing held up better than positional parsing.
- A parser bug made every permit value look like `1`. The fix came from inspecting raw `pdftotext` output, not changing thresholds.
- Deduplication should happen before AI enrichment. It saves cost and keeps reruns predictable.
- Rule-based pre-scoring lets the system archive noise before calling an LLM.
- Docker exposed runtime gaps that local development hid, especially around system binaries like `pdftotext`.
- The system became more resilient by doing less: dates by form, values by behavior, and noisy layout details ignored until needed.

## AI And Quality

- AI can enrich leads, but source evidence should decide whether the lead is trustworthy.
- Every high-priority claim needs evidence, uncertainty, or a reviewer path.
- The project treats AI outreach as a draft for human review, not a send button.
- EvalOps turns production failures into permanent regression cases.
- Local LLMs and provider abstraction matter when privacy, cost, and reliability all count.

## Hotel Sales And Operations

- Construction crews, film crews, professional events, government contracts, and airport disruptions all create hotel demand.
- Creative BC production lists can become lodging signals when filtered for feature films and TV series.
- Convention calendars can separate professional group demand from consumer events.
- YVR disruption monitoring connects aviation signals to revenue operations.
- Slack works for urgent alerts. A web UI works better for review, ownership, and history.

## Product And UI

- The operator UI should serve triage, evidence review, outreach logging, and outcomes before broad analytics.
- Lead detail screens should put AI rationale beside source evidence.
- Reviewer corrections should preserve the original AI value and record who changed it.
- A compact pipeline monitor should answer whether the system is healthy, not replace Grafana.
- Same-origin `/api/*` contracts keep browser access separate from automation tokens.

## Content Pipeline

- A LinkedIn draft pipeline still needs brand voice rules and review gates.
- Good AI content uses concrete facts instead of vague claims.
- Short reviewable drafts beat fully automated publishing.
- Slack review gives operators a lightweight approval layer for AI-generated content.

## Draft Coverage

- Drafts 1 and 2 adapt the two sample posts into the project draft bank.
- Drafts 3 through 7 cover build story and engineering lessons.
- Drafts 8 through 13 cover AI evidence, Slack versus UI, alerts, quality gates, operator workflow, and early demand signals.
- Drafts 14 and 15 cover film and event sources.
- Drafts 16 and 17 cover frontend/API boundaries and model-provider flexibility.
- Drafts 18 through 20 cover the content workflow, small-system architecture, and side-project judgment.
