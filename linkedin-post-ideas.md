# GroupScout LinkedIn Post Ideas

These ideas come from the centralized GroupScout documentation, including the backend README, planning docs, build log, UI strategy, EvalOps plan, and frontend workflow docs.

## Build Story

- The public data was already there. The missing piece was a workflow that read it every week and turned it into sales action.
- Building permits can reveal lodging demand before a contractor sends an RFP.
- GroupScout started with one hotel, one city permit report, and one practical question: which projects will need rooms?
- A hotel sales team can find construction, film, event, bid, and infrastructure leads before competitors see them.
- A small Go service can replace manual weekly scanning across public sources.

## Engineering Lessons

- PDF parsing fails in surprising ways. Content-aware parsing held up better than positional parsing.
- A parser bug made every permit value look like `1`. The fix came from inspecting raw `pdftotext` output, not changing thresholds.
- Deduplication should happen before AI enrichment. It saves cost and keeps reruns predictable.
- Rule-based pre-scoring lets the system archive noise before calling an LLM.
- Docker exposed runtime gaps that local development hid, especially around system binaries like `pdftotext`.

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
