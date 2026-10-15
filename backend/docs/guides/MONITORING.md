# Monitoring And Alerts

IaC Phase 9 (`groupscout-site-6ah.10`). Two layers watch GroupScout:

- **Outside the stack:** a Healthchecks.io dead man's switch expects a ping after every Sun/Tue/Thu lead cadence run.
- **Inside the stack:** Grafana alert rules, provisioned as code, read the app's metrics and Loki logs.

**Status (2026-10-15):** built on backend PR [#15](https://github.com/alvindcastro/groupscout/pull/15), branch `task/iac-phase9-monitoring`. Every CI check passed, and it is waiting for the owner to merge it.
- The live stack runs that build (`31e11db`). Its metrics are seeded from the database, and the five Grafana rules evaluate healthy.
- The external check and the ops Slack channel need the owner's accounts first (see [Owner Setup](#owner-setup)). Until then the heartbeat is off, and Grafana alerts show only in the Grafana UI.

## Why

In July 2026 the Podman move silently stopped the Slack lead deliveries. The Podman machine didn't start at boot, and n8n came up with an empty volume. Nothing noticed for weeks, because nothing expected the cadence to run. A check inside the stack can't catch that kind of failure, since it goes down with the stack. The cadence check therefore runs outside it, and it alerts when a ping *doesn't* arrive.

## What Alerts, And Where

| Alert | Source | Fires when | Goes to |
|---|---|---|---|
| Cadence missed (dead man's switch) | Healthchecks.io check `groupscout-cadence` | No success ping by 18:45 Vancouver on Sun/Tue/Thu, or a `/fail` ping | Healthchecks.io integrations: email to the account by default, plus Slack if you add it there |
| GroupScout API down | Grafana: `up{job="groupscout"} < 1` for 5 min | The API process is down or unreachable | `OPS_SLACK_WEBHOOK_URL` |
| GroupScout lead cadence missed | Grafana: `groupscout_cadence_overdue > 0` | The in-stack backup of the dead man's switch: the latest slot plus 45 min has no completed run | `OPS_SLACK_WEBHOOK_URL` |
| GroupScout collector failed | Grafana: `increase(groupscout_collector_runs_total{result="error"}[2h]) > 0` | A collector errored in a run (one alert per collector) | `OPS_SLACK_WEBHOOK_URL` |
| No new leads in 14 days | Grafana: `time() - groupscout_last_lead_created_timestamp_seconds > 14d` | Six cadence runs produced nothing new | `OPS_SLACK_WEBHOOK_URL` |
| Container restarting repeatedly | Grafana + Loki: more than 3 startup log lines in 15 min for `groupscout_app` or `groupscout_n8n` | A crash loop | `OPS_SLACK_WEBHOOK_URL` |

- **Grafana alerts never use the lead channel.** When `OPS_SLACK_WEBHOOK_URL` is unset, Compose gives Grafana a placeholder URL, so delivery fails harmlessly and the alerts appear only under Grafana > Alerting.
- **n8n's own cadence notices are different.** Those are the "no new leads" and health-preflight messages. They prefer `OPS_SLACK_WEBHOOK_URL` and fall back to `SLACK_WEBHOOK_URL`, as before.

## How The Dead Man's Switch Works

1. n8n triggers `POST /run` with a cadence key on Sun/Tue/Thu at 18:00 Vancouver time.
2. When the run finishes, the API pings `CADENCE_HEARTBEAT_URL`. The ping goes to the URL itself when the run completed, whether or not there were leads, and to `URL/fail` when it errored. A run that found another run holding the lock doesn't ping; the running one does.
3. The Healthchecks.io check has cron schedule `0 18 * * 0,2,4` in `America/Vancouver` with 45 minutes' grace. If no success ping lands in that window, or a `/fail` arrives, it alerts.

So a missing ping covers every way the cadence can fail to happen: n8n stopped, the API down, Podman or the PC off, or the network down. The ping body is the run's summary message. Failure pings carry no error details, because they leave the stack.

`CADENCE_HEARTBEAT_URL` contains the project's ping key. Treat it as a secret: it lives in `prod.sops.env`, and the API never logs it (only the host).

The schedule is defined in three places that must agree, and `internal/smoke/monitoring_test.go` fails if they drift:
- the n8n workflow trigger (`deploy/config/n8n/workflows/sunday-wednesday-lead-cadence.json`);
- `internal/cadence.Default` (Sun, Tue, Thu 18:00 `America/Vancouver`, 45 min grace);
- the check in `deploy/config/healthchecks/checks.json`.

## Metrics

Exported by the API at `/metrics` (`cmd/server/monitoring.go`):

| Metric | Meaning |
|---|---|
| `groupscout_pipeline_runs_total{kind,result}` | Runs by `kind` (`cadence`, `adhoc`) and `result` (`success`, `error`, `locked`, `duplicate`) |
| `groupscout_cadence_last_run_timestamp_seconds` | When the cadence last completed, seeded at startup from `lead_deliveries` (newest row not `failed`) |
| `groupscout_cadence_overdue` | 1 when the latest scheduled slot plus grace has no completed run |
| `groupscout_collector_runs_total{collector,result}` | Collector runs, `ok` or `error`; both series start at 0 for every configured collector |
| `groupscout_collector_last_success_timestamp_seconds{collector}` | Last successful run per collector since the API started |
| `groupscout_leads_created_total` | Leads created by runs and `/ingest` since the API started |
| `groupscout_last_lead_created_timestamp_seconds` | Newest lead, seeded at startup from `leads` |

A fresh install with no history falls back to the API's start time, so it doesn't alert before its first scheduled run. The binary embeds Go's time zone database (`time/tzdata`), because the Alpine image has none. The CI `image` job checks that the gauge reads 0 inside the built image.

## Owner Setup

One-time steps, tracked in `groupscout-site-6ah.16`:

1. **Healthchecks.io account.** Sign up on the free Hobbyist plan and create a project named `GroupScout`.
   - Under Settings > API Access, create a **read-write API key** and a **ping key**.
   - Check that the email integration is on (Integrations), and add Slack there too if you like.
2. **Add the secrets with `make secrets-edit`:**
   - `HEALTHCHECKS_API_KEY=<read-write key>`. Only `make monitors-apply` uses it; no container receives it.
   - `CADENCE_HEARTBEAT_URL=https://hc-ping.com/<ping key>/groupscout-cadence`.
   - `OPS_SLACK_WEBHOOK_URL=<incoming webhook for a new #groupscout-ops channel>`.
3. **Render and apply.** Run `make secrets-render FORCE=1`, then `make monitors-apply`. That creates the check with the schedule above, and rerunning it updates it in place.
4. **Recreate the services** that read the new values: `podman compose --profile obs up -d groupscout grafana n8n`. After the cutover to the WSL host (`groupscout-site-6ah.18`), run `make ansible-apply` instead; it renders the new `.env` and recreates what changed. The API logs `cadence heartbeat enabled host=hc-ping.com`.
5. **Test it.** The check shows "new" until the next cadence slot, then "up" after the run's ping. To test the alert without missing a real run, pause a test check, or use Healthchecks' "Send test notification" on the integration.

## Changing Things

- **Alert rules.** Edit `deploy/config/grafana/provisioning/alerting/rules.yml`, run `go test ./internal/smoke/`, then `podman compose --profile obs up -d --force-recreate grafana`. Provisioned rules are read-only in the Grafana UI.
- **The schedule.** Change the n8n workflow, `cadence.Default` and `checks.json` together, then run `make monitors-apply`.
- **Grafana 13's Slack integration** needs the webhook to answer exactly `ok`. A mock that answers anything else makes Grafana treat the firing notice as failed, and it then never sends the matching "resolved" message.

## Verification (2026-10-15)

- **Simulated on an isolated Podman network:** the Phase 9 image, a seeded Postgres, Prometheus, Grafana 13.2.4 with this provisioning, and a mock receiver for Slack and Healthchecks.
  - With the last run on Sunday, `groupscout_cadence_overdue` read 1. "GroupScout lead cadence missed" reached the mock ops Slack.
  - A cadence `POST /run` sent the success ping with the run summary and cleared the gauge. Grafana then sent "[RESOLVED] GroupScout lead cadence missed".
  - With the database stopped, a cadence run returned 500 and sent `/fail` with a generic body.
  - Stopping the API container fired "GroupScout API down" after 5 minutes.
- **`make monitors-apply` against a mock of the Healthchecks v3 API:** the first run creates the check and a second run updates it rather than duplicating it. A wrong or missing key fails.
- **Live:**
  - The API seeded the last run (Tue 2026-10-13 18:00:19) and newest lead from Postgres, and reports `groupscout_cadence_overdue 0`.
  - The live Grafana loaded all five rules and evaluates them healthy against live Prometheus and Loki.
- **Deferred:**
  - An external uptime check on a public `/health` needs public ingress and an always-on host (Phase 6). A home PC that sleeps at night would page every night.
  - The Postgres disk alert needs host metrics (node_exporter, Phase 5).
  - ~~The `obs` profile isn't started by the Windows autostart task.~~ Fixed in Phase 10: `deploy/windows/podman-autostart.ps1` starts Prometheus, Loki, Alloy and Grafana at logon. The external check never depended on it.

## Related

- Plan: [INFRASTRUCTURE_AS_CODE.md](../planning/INFRASTRUCTURE_AS_CODE.md) Phase 9
- Secrets: [SECRETS.md](./SECRETS.md)
- n8n cadence workflow: [N8N_GUIDE.md](./N8N_GUIDE.md)
- CI: [CI.md](./CI.md)
