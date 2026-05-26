# GroupScout LinkedIn Post Drafts

These drafts use concise, active, concrete writing. They avoid em dashes and avoid one-sentence paragraph style.

## Post 1: Public Data Becomes Sales Action

Public data does not create value by itself. Someone still has to read it, filter it, and act before the opportunity gets cold.

That was the problem behind GroupScout. Richmond building permits, Creative BC production lists, VCC events, CivicInfo bid feeds, and infrastructure announcements all contain useful hotel sales signals.

The system watches those sources, deduplicates the raw records, filters out noise, enriches promising leads, and sends the best ones to the sales team. The goal is simple: find group lodging demand before it becomes an inbound RFP.

#HotelSales #Automation #BuiltDifferent

## Post 2: The Parser Bug

One bug made every construction permit look worthless. The parser read a bare `1` as the construction value and missed the real `$300,000.00` line below it.

The fix was not a better threshold. The fix was reading the raw `pdftotext` output and parsing fields by content: dates looked like dates, dollar values looked like dollar values, and folder names carried addresses.

That lesson stuck. Real-world data rewards parsers that understand shape and meaning, not line numbers.

#DataEngineering #GoLang #Automation

## Post 3: Dedup First, Enrich Second

AI calls should not be the first step in a pipeline. GroupScout stores raw records and checks a deterministic hash before it spends money on enrichment.

That choice makes reruns calm. If the process fails halfway through, the raw input still exists, and the next run can continue without creating duplicates.

It also keeps costs honest. The system should know whether a record is new before it asks an LLM to analyze it.

#AIEngineering #DataPipelines #GoLang

## Post 4: Rules Before AI

GroupScout does not send every public record to an LLM. A Go pre-scorer filters small residential renovations, minor repairs, and weak signals before enrichment starts.

That rule-based layer matters. It saves tokens, reduces noise, and makes the AI work on leads that may create real room-night demand.

The LLM adds judgment after the system has already done basic discipline. Good AI systems often start with ordinary software rules.

#AI #Automation #HotelSales

## Post 5: Evidence Beside AI

An AI score is not enough for a sales rep. The rep needs to know which source produced the lead, what the raw evidence said, and why the system thinks timing matters.

That is why the planned operator UI puts source evidence beside AI rationale. Reviewer corrections preserve the original AI value instead of silently replacing it.

This design keeps trust visible. A lead can be promising, uncertain, corrected, or dismissed, but the system should show how it reached that state.

#ProductDesign #AI #SalesOps

## Post 6: Slack Is An Alert, Not A Workspace

Slack is useful when something needs attention now. It is weak when a team needs ownership, source review, outreach history, and outcome tracking.

GroupScout keeps Slack as the interrupt channel. High-priority leads and airport disruption alerts can still land there first.

The web UI has a different job. It gives the sales team a durable place to triage leads, review evidence, claim ownership, log outreach, and measure outcomes.

#ProductManagement #SalesOps #HotelTech

## Post 7: Airport Disruptions Are Demand Signals

Airport disruption monitoring is not just an operations problem. For an airport hotel, flight cancellations, weather alerts, and NOTAMs can signal sudden room demand.

GroupScout's `alertd` watches YVR conditions and computes a Stranded Passenger Score. It routes urgent alerts to the right Slack channel and updates the same message as conditions change.

The product lesson is broader than aviation. Alert systems should reduce confusion, not add channel noise.

#RevenueOps #Hospitality #Automation

## Post 8: AI Quality Needs Gates

AI quality cannot depend on vibes. GroupScout documents release gates for unsupported high-priority claims, fabricated sources, unsafe outreach, stale alert data, and secret leakage.

The EvalOps plan uses golden cases, deterministic scorers, Promptfoo-compatible checks, and review samples. Production failures can become new fixtures after a human reviews them.

That feedback loop matters. If an AI system can make the same mistake twice, the second failure should become a test.

#AIEval #QualityEngineering #Automation

## Post 9: Operator UI Before Analytics

A dashboard is tempting, but the first UI should help people do the work. For GroupScout, that means lead triage, evidence review, status changes, outreach notes, and owner assignment.

Analytics still matters. Source yield, aging leads, won and lost outcomes, and demand timing become useful once the workflow captures reliable activity.

The order matters. Build the operating surface first, then measure the work it produces.

#UXDesign #ProductStrategy #SalesOps

## Post 10: Content Drafts Still Need Rules

AI can draft LinkedIn posts, but the draft still needs a standard. The LUX content workflow uses concrete facts, short copy, review in Slack, and a ban on filler openings.

That approach fits GroupScout too. A useful post should name the problem, show the detail, and leave the reader with one clear point.

Automation can speed up the first draft. It should not remove judgment from publishing.

#ContentOps #AI #BuiltDifferent

## Post 11: A Small System Can Be Serious

GroupScout started as a focused tool for one hotel sales workflow. It now has collectors, scoring, enrichment, Slack delivery, email digests, Postgres storage, audit trails, Docker deployment, and an operator UI plan.

The system grew because the interfaces stayed simple. A source becomes a collector, a record becomes a raw project, a qualified project becomes a lead, and a lead becomes a sales action.

That is the kind of architecture I trust. It stays close to the work and earns complexity only when the workflow needs it.

#SoftwareArchitecture #GoLang #Automation

## Post 12: The Best Lead Is Early

A hotel sales lead loses value when every competitor already knows about it. Construction starts, film productions, conference schedules, bid awards, and infrastructure announcements all appear before many formal buying conversations.

GroupScout looks for those signals early. It estimates crew size, lodging need, outreach timing, and property fit before the lead reaches a traditional market report.

That shift changes the sales motion. The team stops waiting for demand and starts scouting for it.

#HotelSales #MarketIntelligence #Automation
