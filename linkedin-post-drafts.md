# LinkedIn Post Drafts

These drafts use the prior LinkedIn posts as the baseline voice:

- introduce the work as "my side project, GroupScout" when the name helps
- use "the system," "the project," or "this side project" after the first reference
- first person where the post is about build judgment
- concrete system details before broad claims
- one engineering lesson per post
- short paragraphs that still carry a complete thought
- no hype, no vague AI language, no fully automated publishing claims

## Draft 1: First End-to-End Pipeline

I built the first end-to-end pipeline for my side project, GroupScout.

The first version was intentionally narrow:

1. Fetch a PDF from a static URL.
2. Extract text with `pdftotext`.
3. Regex for the first five projects.
4. Send the project text to Claude for scoring.
5. Push a JSON payload to Slack.

It was fragile. It failed when the PDF was missing. It failed when Claude returned malformed JSON. It had no real retry story and no durable state.

But it proved the important thing. A lead could move from a municipal PDF into a channel where a sales team could see it.

That was the milestone. Not scale, not architecture, not elegance. Visibility.

Once a lead appeared in Slack, the project stopped being an idea and became a system worth hardening.

Engineering breakdown: https://lnkd.in/gb6Eqxhq

## Draft 2: Where Uncertainty Belongs

A follow-up to my earlier posts about this side project.

Once the basic pipeline worked, the hard part changed. The problem was no longer whether public data could become a lead. It was where uncertainty should live.

Municipal PDFs were less predictable than the demo path suggested. Encodings varied, layouts shifted, and lightweight libraries failed quietly.

The risk was not missing features. The risk was absorbing variability in the wrong layer and paying for it through fragile application code.

Because this was a side project, I had to make that tradeoff early. Which problems belonged in my code? Which belonged in tools that have handled document edge cases for decades? How much instability should the system tolerate by design?

That led to a move away from layout-dependent parsing and toward content-based reasoning. Dates are identified by form, values by behavior, and everything else is treated as noise unless it earns trust.

The system became more resilient by doing less.

Technical note: https://lnkd.in/gTfu57ER

## Draft 3: Public Data Is Not A Workflow

Public data does not create value by itself.

Richmond building permits, Creative BC production lists, CivicInfo bid feeds, event calendars, and infrastructure announcements all contain useful hotel sales signals. They are also scattered, inconsistent, and easy to ignore when someone has a full week of real work already.

That was the operating problem behind my side project, GroupScout.

The system watches public sources, stores raw records, filters noise, enriches promising leads, and sends the best ones to the sales team. The goal is not to collect more data. The goal is to make public signals usable before they turn into inbound RFPs.

The distinction matters. A source is only useful if it becomes a timely action.

#HotelSales #Automation #DataEngineering

## Draft 4: The Parser Bug

One bug made every construction permit look worthless.

The parser read a bare `1` as the construction value and missed the real `$300,000.00` line below it. On paper, the scoring logic looked strict. In practice, the input was wrong.

The fix did not come from changing the threshold. It came from reading the raw `pdftotext` output and accepting that the document structure was not stable enough to trust by position.

Dates needed to look like dates. Dollar values needed to behave like dollar values. Folder names carried addresses. The parser had to care about content, not just line numbers.

That lesson shaped the rest of the system.

Real-world data does not reward clever assumptions. It rewards boring checks that keep working after the layout changes.

#DataEngineering #GoLang #Automation

## Draft 5: Dedup Before AI

AI calls should not be the first step in a pipeline.

In this side project, raw records are stored and checked with a deterministic hash before enrichment runs. That sounds like plumbing, but it changes how calm the system feels.

If a run fails halfway through, the raw input still exists. If the collector runs again, duplicate records do not become duplicate leads. If the LLM provider is slow or unavailable, the source evidence is not lost.

It also keeps cost honest.

The system should know whether a record is new before it asks an LLM to analyze it.

That order is easy to skip in a prototype. It is also the difference between a demo and something you can rerun without worrying about what it will damage.

#AIEngineering #DataPipelines #GoLang

## Draft 6: Rules Before AI

The system does not send every public record to an LLM.

A Go pre-scorer filters small residential renovations, minor repairs, weak permit signals, and obvious non-leads before enrichment starts.

That layer is not glamorous, but it matters. It saves tokens. It reduces noise. It keeps the model focused on records that may actually create room-night demand.

The LLM adds judgment after the system has already done basic discipline.

That has become one of my rules for practical AI systems. Use ordinary software to remove the obvious cases first. Then use AI where the ambiguity is real.

#AI #Automation #HotelSales

## Draft 7: Docker Found The Real Dependency

Local development made the PDF pipeline look more self-contained than it was.

Then Docker exposed the truth. The code depended on `pdftotext`, and that meant the runtime image needed the right system package installed, not just the Go binary.

That is a small detail, but it is exactly the kind of detail that separates a local script from an operational service.

The fix was simple once the failure was visible. The better lesson was broader: system boundaries should include the tools your code shells out to.

If a parser depends on a binary, the deployment contract depends on that binary too.

#Docker #DataEngineering #GoLang

## Draft 8: Evidence Beside AI

An AI score is not enough for a sales rep.

The rep needs to know which source produced the lead, what the raw evidence said, and why the system thinks timing matters. Without that context, the score becomes another black box to ignore.

That is why the planned operator UI puts source evidence beside AI rationale. Reviewer corrections preserve the original AI value instead of silently replacing it.

The design goal is trust through visibility.

A lead can be promising, uncertain, corrected, or dismissed. The system should show how it reached that state and who changed it afterward.

#ProductDesign #AI #SalesOps

## Draft 9: Slack Is An Alert, Not A Workspace

Slack was the right place for the first lead from this side project to appear.

It made the pipeline visible. It created momentum. It proved the system could move from public data to a sales action.

But Slack is not the right place for everything.

Ownership, evidence review, outreach history, status changes, and outcomes need a durable workspace. A message thread cannot carry the full lifecycle of a lead without becoming noisy or incomplete.

That is how the product boundary became clearer.

Slack is the interrupt channel. The web UI is the operating surface.

#ProductManagement #SalesOps #HotelTech

## Draft 10: Airport Disruptions Are Demand Signals

Airport disruption monitoring is not only an operations problem.

For an airport hotel, cancellations, weather alerts, ground delays, and NOTAMs can signal sudden room demand. The sales and operations teams need to see those signals before they become scattered updates across multiple sites.

The `alertd` service watches YVR conditions and computes a Stranded Passenger Score. Urgent alerts can land in Slack, and updates can modify the same message as conditions change.

The product lesson is broader than aviation.

An alert system should reduce confusion. If it creates more channels to watch, it is not done yet.

#RevenueOps #Hospitality #Automation

## Draft 11: AI Quality Needs Release Gates

AI quality cannot depend on whether a demo looked good.

The project documents release gates for unsupported high-priority claims, fabricated sources, unsafe outreach, stale alert data, and secret leakage. Those gates are not academic. They describe the failures that would make the system less trustworthy in real use.

The EvalOps plan uses golden cases, deterministic scorers, Promptfoo-compatible checks, and review samples. Production failures can become new fixtures after a human reviews them.

That feedback loop matters.

If an AI system can make the same mistake twice, the second failure should become a test.

#AIEval #QualityEngineering #Automation

## Draft 12: Operator UI Before Analytics

A dashboard is tempting because it looks like progress.

For this side project, the first UI needs to help people do the work: triage leads, inspect evidence, change status, assign an owner, and log outreach.

Analytics still matters. Source yield, aging leads, won and lost outcomes, and demand timing become useful once the workflow captures reliable activity.

But the order matters.

If the operating surface is weak, the analytics will mostly measure confusion. Build the workflow first, then measure the work it produces.

#UXDesign #ProductStrategy #SalesOps

## Draft 13: The Best Lead Is Early

A hotel sales lead loses value when every competitor already knows about it.

Construction starts, film productions, conference schedules, bid awards, and infrastructure announcements all appear before many formal buying conversations. Those signals are not clean, but they are early.

The system looks for that early shape of demand. It estimates crew size, lodging need, outreach timing, and property fit before the lead reaches a traditional market report.

That changes the sales motion.

The team stops waiting for demand to announce itself and starts scouting for it while it is still forming.

#HotelSales #MarketIntelligence #Automation

## Draft 14: Creative BC As A Lodging Signal

A film production list is not a hotel lead by default.

Most entries need filtering. A feature film or TV series can imply crew lodging, extended stays, production offices, and changing room blocks. A small or irrelevant listing may not matter at all.

That is why the project treats Creative BC as a signal source, not a finished answer.

The collector can capture the public record. Rules can filter obvious noise. AI can help reason about project type, timing, and likely lodging fit. A human can still decide whether the lead deserves outreach.

The value comes from the chain, not any single step.

#FilmProduction #HotelSales #Automation

## Draft 15: Convention Calendars Need Judgment

Not every event creates group lodging demand.

Some events are local. Some are consumer-facing. Some are too small, too short, or too close to matter for hotel sales. Others are exactly the kind of professional gathering that should trigger timely outreach.

That is the reason the project does not treat event calendars as simple scrape-and-send sources.

The useful work is classification. What type of event is it? Who attends? How far away is it? Does it fit the property? Is there enough lead time for a sales rep to act?

Good automation does not remove judgment. It moves judgment to the places where it creates leverage.

#EventSales #Hospitality #Automation

## Draft 16: Same-Origin Browser Contracts

The frontend plan uses same-origin `/api/*` routes for browser access.

That may sound like infrastructure detail, but it keeps a useful boundary in place. Browser sessions and automation tokens should not be treated as the same thing.

The backend can serve operators, collectors, workflow tools, and integrations. The frontend should get the access pattern meant for humans using the product in a browser.

Small architecture decisions like this reduce future cleanup.

Auth and deployment choices become harder to unwind after the UI grows around them.

#WebArchitecture #Security #ProductEngineering

## Draft 17: Local LLMs Change The Tradeoff

This side project has to balance privacy, cost, latency, and reliability.

That is why provider abstraction matters. Some enrichment tasks may fit a hosted model. Some may be better suited to a local model through Ollama, especially when cost or data sensitivity becomes the constraint.

The point is not to make the system model-agnostic for its own sake.

The point is to keep the business workflow from being trapped behind one provider decision. Lead scoring, evidence summaries, outreach drafts, and quality checks should be able to evolve as the model landscape changes.

AI systems need product flexibility as much as prompt quality.

#LLMOps #AIEngineering #Automation

## Draft 18: The Content Pipeline Needs Judgment Too

I am using the same lesson from building this side project on the content side.

AI can help draft LinkedIn posts, but the system still needs a standard. The draft has to name the problem, include concrete facts, avoid filler, and make one clear point.

That is why the content workflow treats AI output as a draft for review, not a publish button.

Slack review is enough for lightweight approval. A human still decides whether the post says something true and useful.

Automation should make judgment cheaper to apply. It should not remove judgment from the process.

#ContentOps #AI #FounderBuild

## Draft 19: Small Systems Can Be Serious

My side project, GroupScout, started with one hotel sales workflow.

It now has collectors, scoring, enrichment, Slack delivery, email digests, Postgres storage, audit trails, Docker deployment, and an operator UI plan.

That sounds like a lot, but the core shape stayed simple. A source becomes a collector. A record becomes raw input. A qualified record becomes a lead. A lead becomes a sales action.

That is the kind of architecture I trust.

It stays close to the work and earns complexity only when the workflow needs it.

#SoftwareArchitecture #GoLang #Automation

## Draft 20: Side Projects Expose Weak Assumptions

Side projects are useful because weak assumptions surface quickly.

There is no large team to absorb ambiguity. No process to hide behind. No stakeholder meeting that turns a technical risk into someone else's problem.

In this project, the weak assumptions appeared in the boring places: PDF layout stability, missing binaries, malformed model output, duplicate records, and unclear ownership after a Slack alert.

Each one forced a decision.

Fix it in application code, move the responsibility to a better tool, add a release gate, or reduce scope until the system is honest again.

That is why I keep building side projects. They make engineering judgment unavoidable while the cost of being wrong is still small.

#SideProjects #Engineering #BuildInPublic

## Draft 21: One Raw Item Should Not Start The Whole Machine

The project needed a smaller door into the pipeline.

The full collector run still matters. It scans public sources, filters noise, enriches records, stores leads, and sends alerts.

But n8n sometimes finds one useful item. Starting every collector for one raw project would waste time and blur responsibility.

That is why I added `POST /ingest`.

The endpoint accepts one raw project or event payload. It uses the same audit, dedup, scoring, enrichment, and storage path as the full system. It does not direct-insert a finished lead. It does not skip the evidence trail.

That distinction matters.

A small input path should still obey the same rules as the larger system.

#DataPipelines #Automation #GoLang

## Draft 22: Idempotency Is A Product Feature

The Sunday and Wednesday lead cadence had a simple promise: send one eligible lead.

That promise needed more than a cron schedule.

Retries happen. Containers restart. n8n can run a workflow again. Slack can receive the same alert twice if the backend treats each request as new.

The fix was a cadence key.

`lead-cadence:2026-05-27:wednesday` became both a schedule marker and an idempotency key. The backend records delivery status, protects the run with a lock, and returns `sent`, `duplicate`, `locked`, or `no_eligible_lead`.

The user-facing result is plain.

One cadence should create one send.

#BackendEngineering #Reliability #Automation

## Draft 23: A Health Check Should Fail Before The Real Work

The n8n workflow now checks backend health before it runs the lead cadence.

That saved time during a real failure.

The app had a stale Docker image. Postgres had already reached migration version `10`; the image only contained migrations through `009`. The backend exited before it could serve `/health`.

n8n did the right thing. It stopped before calling `/run` and sent an ops Slack message.

The alert looked small: `status=0`.

The cause was concrete: no HTTP response, because the backend was not running.

Preflight checks are useful when they point at the broken layer before the main workflow does damage.

#DevOps #Reliability #Docker

## Draft 24: Keep The Evidence, Bound The Storage

Raw audit data is useful until it becomes unmanaged storage.

The project stores source payloads because sales leads need evidence. A score without the original PDF, API body, URL, or collector name is hard to trust.

But raw data also grows.

The retention worker solves the narrow problem. It can purge old raw inputs on an opt-in schedule, but it preserves any payload still referenced by a lead.

That rule keeps the system honest.

Delete stale evidence. Keep live evidence. Make the operator choose the policy.

#DataGovernance #BackendEngineering #ProductOps

## Draft 25: Login Is Part Of The Product, Not A Wrapper

The admin login flow needed to do more than accept a token.

A browser user needed a clear path: read the current setup token, submit it, get a session cookie, verify the session, use the app, and log out.

The backend also needed to rotate file-backed setup tokens after use and reject stale tokens.

That work is easy to treat as plumbing. It is not.

Authentication shapes every route that follows it. If protected screens render before the session check, the product teaches the wrong boundary.

The system should make the secure path the ordinary path.

#WebSecurity #ProductEngineering #BackendEngineering

## Draft 26: Browser APIs Need Their Own Boundary

The browser should not know the automation token.

That rule shaped the UI API work.

The backend can expose server-to-server routes for n8n, cron jobs, and operators with bearer tokens. The web UI needs same-origin `/api/*` routes protected by browser sessions.

Those are different clients. They deserve different contracts.

The operator UI now has paths for leads, raw evidence, outreach, pipeline runs, stats, system health, and alerts. The frontend calls them through same-origin adapters instead of passing backend secrets into browser code.

Small boundaries prevent large leaks.

#WebArchitecture #Security #ProductEngineering

## Draft 27: Outreach Needs State, Not A Text Box

A lead workflow breaks down when every update becomes a note.

The operator needs constrained actions: claim, dismiss, snooze, flag, contacted, won, lost, no response, and reopen. The system needs to know which actions are valid from the current state.

That is why outreach moved into storage and API contracts.

The backend records outreach history. It validates status transitions. It returns conflicts when a request asks for an impossible move.

The UI can still feel simple.

The discipline lives under it.

#SalesOps #ProductEngineering #BackendEngineering

## Draft 28: Run History Beats A Spinner

Long-running pipeline work should not trap the browser.

The backend now accepts a pipeline run request, records the run, starts work without blocking the operator, and exposes run history through `/api/pipeline/runs`.

That gives the UI a better contract.

The operator can see whether a run started, failed, or finished. The browser does not need to hold an open request and hope nothing times out.

A spinner explains nothing.

A run record explains what happened.

#BackendEngineering #UX #Automation

## Draft 29: A Dashboard Needs A Denominator

Analytics can mislead when they skip the denominator.

The GroupScout UI models status counts, source yield, score bands, owner workload, lead aging, verification quality, and upcoming demand. Each number needs context: date range, total leads, source count, and what the system excluded.

That work is less flashy than drawing charts.

It matters more.

A chart without a clear denominator invites a false conclusion. A compact table with plain labels can help the operator decide what to do next.

Measure the workflow after the workflow records enough truth.

#Analytics #ProductDesign #SalesOps

## Draft 30: The UI Started As A Runtime Contract

The first production UI did not start with a framework.

It started with a tested runtime contract.

Serve `web/dist`. Expose `/healthz`. Proxy browser `/api/*` requests server-side. Keep provider keys, Slack tokens, database URLs, session secrets, and automation tokens out of browser-visible assets.

That constraint kept the work narrow.

The UI could gain routes, render screens, and join Docker Compose without changing the security posture.

Small runtime rules do not slow a product down. They keep later work from starting in debt.

#FrontendEngineering #WebSecurity #Docker

## Draft 31: Split The Client Before It Becomes A Dumping Ground

The browser API client was starting to carry too much.

Routes, payload builders, response adapters, transport rules, and feature methods all lived too close together.

The refactor kept the public facade and split the internals by feature: leads, raw audit, outreach, pipeline, stats, alerts, and system.

The important part was what did not change.

The browser still imports `createApiClient(...)`. Same-origin transport still rejects unsafe paths. Feature code now changes in the module that owns it.

Good refactors preserve the useful contract and remove the unnecessary crowding.

#Refactoring #FrontendEngineering #SoftwareDesign

## Draft 32: The Command Center Should Show Work, Not Decoration

The Today command center has a simple job.

It should show the operator what needs attention now: priority leads, aging work, active alerts, failed jobs, and system health.

That is different from a landing page.

An operations screen should be dense, quiet, and easy to scan. It should link to the right workspace and avoid pretending that every summary is an action.

The system already has alerts, analytics, lead detail, verification, and outreach screens. The command center should connect those surfaces.

The first screen should help the operator start work.

#UXDesign #ProductEngineering #Operations

## Draft 33: The Work Is In The Edges

The happy path rarely proves much.

The useful engineering work showed up in the edges: duplicate cadence runs, stale Docker images, missing migrations, malformed model output, raw evidence retention, session expiry, and browser routes that should never see automation tokens.

None of those problems looks impressive in isolation.

Together, they decide whether a system can run twice without making people nervous.

That is where I try to spend time on this side project.

Make the normal path clear. Make the failure path visible. Make reruns boring.

#SoftwareEngineering #Reliability #BuildInPublic

## Draft 34: Tests Should Guard Decisions, Not Just Code

I do not want tests that only prove a function returns the expected value today.

I want tests that guard a decision.

The UI Docker tests say browser code must not receive backend secrets. The API client tests say browser requests stay same-origin. The cadence tests say one schedule key cannot send the same lead twice. The storage tests say raw evidence referenced by a lead must survive retention cleanup.

Those are product and architecture choices.

The code can change. The boundary should not drift by accident.

Good tests make the important decisions hard to erase.

#Testing #SoftwareDesign #ProductEngineering

## Draft 35: A Small System Still Needs Operating Discipline

GroupScout is a side project, but I do not want it to behave like a script.

A script can fail and ask the operator to remember what happened.

A system should leave evidence.

That is why the project records raw inputs, pipeline runs, delivery status, outreach history, session state, and health responses. Each record answers a question someone will ask later.

Did this run start? Did it send? Did it duplicate? Which source produced this lead? Who changed the status? Can n8n reach the backend?

Operating discipline starts before scale.

#BackendEngineering #Operations #Reliability

## Draft 36: Product Sense Shows Up In What You Refuse To Automate

The project could auto-send outreach.

I chose not to.

A lead can be scored, enriched, summarized, and drafted by software. The final message still needs human review because the cost of a wrong outreach is not just technical.

That choice shapes the product.

The system can prepare context, show evidence, suggest timing, and record actions. It should not pretend that every qualified signal deserves an automatic email.

Useful automation reduces the cost of judgment.

It does not remove judgment where judgment protects trust.

#ProductEngineering #AI #SalesOps

## Draft 37: Refactoring Is Easier When The Contract Is Clear

The frontend API client grew too large.

That was not a reason to rewrite it.

The public contract still worked: `createApiClient(...)`, same-origin requests, feature methods, normalized payloads, and safe response adapters.

So the refactor kept the contract and moved the weight behind it.

Leads, raw audit, outreach, pipeline, stats, alerts, and system calls now live in focused modules. The shared transport guard stayed central.

The lesson is simple.

Do not refactor around irritation. Refactor around a boundary you can name and preserve.

#Refactoring #FrontendEngineering #SoftwareArchitecture

## Draft 38: Documentation Should Reduce The Next Failure

I have been moving GroupScout docs into the coordination repo.

That sounds like cleanup. It is more practical than that.

The n8n cadence runbook now explains the health preflight, required environment variables, workflow activation checks, and guaranteed delivery payload. The Docker docs separate UI-only tests from backend-dependent smoke checks. The troubleshooting notes name current container names and known caveats.

Documentation should not merely describe the system.

It should shorten the next recovery.

When something fails, the runbook should tell you where to look first.

#Documentation #DevOps #SoftwareEngineering

## Draft 39: Secure Defaults Are Product Work

The UI runtime has one rule I care about: browser code does not get server secrets.

That rule affects Docker, the production server, API routing, tests, and docs.

The production server serves static assets and proxies `/api/*` requests server-side. The UI uses same-origin calls. The tests scan for forbidden public config. The docs warn against passing backend `.env` into frontend containers.

This is not security theater.

It is a product boundary.

Once a secret leaks into browser-visible code, every feature built on top inherits the mistake.

#WebSecurity #FrontendEngineering #ProductEngineering

## Draft 40: The Best Debugging Artifact Was The Exact Error

The n8n alert said `status=0`.

That was not enough.

The useful clue came from the backend log: `no migration found for version 10`. Postgres had version `10`; the Docker image only had migration files through `009`.

That narrowed the fix.

No rewrite. No workflow redesign. Rebuild the image, recreate the backend container, verify `/health` from host and from inside n8n.

Good debugging turns a vague symptom into a concrete mismatch.

Then the fix gets smaller.

#Debugging #Docker #Reliability
