# Phase 7 Pipeline Monitor And Run Controls

Phase 7 adds a compact operator-facing Pipeline Monitor. It answers whether the lead pipeline is fresh and healthy, while leaving deep observability to logs and Grafana.

## Scope

- `web/src/app/pipelineMonitor.js` owns the Pipeline Monitor screen model, mocked run data, compact health summaries, recent failure summaries, run history rows, responsive metadata, and run-control action metadata.
- `createRouteShell("/pipeline")` mounts the Pipeline Monitor instead of a placeholder.
- `createApiClient().listPipelineRuns(...)` reads recent run history through same-origin `GET /api/pipeline/runs`.
- `createApiClient().startPipelineRun(...)` starts a manual run through same-origin `POST /api/pipeline/runs`.

## Health Fields

- Freshness: latest run time and result.
- Collector: collected, skipped, enriched, and recent failure messages.
- LLM: provider, latency in milliseconds, and error count.
- Delivery: Slack, email, and webhook delivery failures.
- Observability: optional logs and Grafana links when supplied by the API.

## Run-Control Behavior

- Manual run creation is browser-async: the UI receives queued/running metadata and does not wait for the pipeline to finish.
- The screen exposes only `POST /api/pipeline/runs` as its run-control endpoint.
- Automation-only endpoints such as one-shot `/run`, worker internals, scheduler hooks, and webhook delivery URLs remain out of browser components.

## Failure And Partial Data

- Loading, empty, and error states are represented explicitly.
- Partial data stays renderable when optional LLM, notification, or collector detail is missing.
- Missing observability links do not block health rendering.

## Design Tokens

Phase 7 reuses existing documented `DESIGN.md` component tokens:

- `card-base` for compact health and history surfaces.
- `button-primary` for the manual run control.
- `button-secondary` for secondary refresh-style controls.
- `badge-tag` for compact health labels.
- `property-row` for run history rows.

No new `DESIGN.md` token names were required.

## TDD Evidence

- Red run: `npm test` failed because `createApiClient().listPipelineRuns(...)`, `createApiClient().startPipelineRun(...)`, and `web/src/app/pipelineMonitor.js` did not exist.
- Targeted green runs: `node --test test/pipeline-client.test.js` and `node --test test/pipeline-monitor.test.js`.
- Full-suite green run: `npm test`.

## Out Of Scope

- Replacing Grafana.
- Rendering log bodies inline.
- Direct browser calls to automation-only endpoints.
- Worker polling or blocking the browser until a long-running pipeline completes.
- A custom dashboard builder or analytics suite.
