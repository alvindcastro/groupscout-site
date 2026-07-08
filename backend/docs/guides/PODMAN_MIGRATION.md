# Podman Migration Runbook

This runbook captures the current Docker-to-Podman migration review for GroupScout. Docker remains the known-good runtime until the checks below are run against the backend and UI source repos; use this file to keep the migration explicit instead of scattering one-off command substitutions through older Docker docs.

Run backend runtime commands from `/mnt/c/Users/alvin/GolandProjects/groupscout` and UI runtime commands from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

## Current Decision

- Docker Compose is still the baseline documented runtime.
- Podman support is validated for local development and smoke testing. Docker Compose remains the production baseline until deployment docs, CI, and production socket/logging assumptions are updated together.
- 2026-07-01 verification: `RedHat.Podman` 5.8.3 and standalone `Docker.DockerCompose` 5.2.0 are installed. `podman machine init` and `podman machine start` provisioned a rootless WSL-backed machine, `podman --version`, `podman compose version`, and `podman info` passed, the backend stack started, the UI test image passed, and the backend-owned UI E2E smoke passed under Podman.
- Current Windows notes: PowerShell needs the Winget user PATH refreshed for `docker-compose.exe` after install. Git Bash can run the backend-owned smoke script with `GROUPSCOUT_COMPOSE="podman compose"` when the Docker Compose Winget package directory is on PATH. Plain WSL has `make`, but cannot execute the Windows `podman.exe` in this environment.
- Prefer runtime variables in local notes and scripts:

```sh
COMPOSE="docker compose"      # known-good baseline
COMPOSE="podman compose"      # migration candidate
CONTAINER_CLI="docker"        # direct container CLI
CONTAINER_CLI="podman"
```

Use service-oriented Compose commands where possible:

```sh
$COMPOSE ps
$COMPOSE logs groupscout --tail=50
$COMPOSE exec postgres psql -U groupscout -d groupscout -c "SELECT 1;"
$COMPOSE exec ollama ollama list
```

Avoid new runbook commands that depend on fixed container names such as `groupscout_postgres` unless the command is explicitly marked Docker-tested. The backend Compose file currently sets fixed `container_name` values, but service names are the more portable boundary.

## Migration Smoke Sequence

1. Confirm Podman and Compose availability:

```sh
podman --version
podman compose version
podman info
```

2. Validate the backend Compose file:

```sh
COMPOSE="podman compose"
cd /mnt/c/Users/alvin/GolandProjects/groupscout
$COMPOSE -p groupscout config --quiet
```

3. Start the core backend stack and check health:

```sh
$COMPOSE -p groupscout up -d --build postgres groupscout n8n
$COMPOSE -p groupscout ps
curl -i --max-time 5 http://localhost:8080/health
$COMPOSE -p groupscout exec postgres psql -U groupscout -d groupscout -c "SELECT 1;"
```

4. Check logs without Docker-only container names:

```sh
$COMPOSE -p groupscout logs --tail=100 groupscout postgres n8n ollama
```

5. Validate the UI test image:

```sh
cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui
podman build --target test -t groupscout-ui-test .
podman run --rm groupscout-ui-test
```

6. Validate merged backend plus UI Compose config:

```sh
COMPOSE="podman compose"
$COMPOSE -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  config --quiet
```

7. Smoke the development UI product server:

```sh
$COMPOSE -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f compose.dev.yml \
  up --build groupscout-ui groupscout

curl -i --max-time 5 http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/healthz
curl -i --max-time 5 http://localhost:${GROUPSCOUT_UI_HOST_PORT:-3001}/
```

8. Smoke the D4 production UI image in standalone mode:

```sh
podman build --target production -t groupscout-ui-production .
podman run --rm -p 3002:3000 \
  -e UI_API_PROXY_TARGET=http://host.containers.internal:8080 \
  groupscout-ui-production
```

If `host.containers.internal` is unavailable in the local Podman setup, use a backend container on the same Compose network and the service target `http://groupscout:8080` instead. Browser-visible UI code must still use relative `/api/*`.

9. Run the backend-owned UI E2E gate after the source repo has Podman-compatible scripts:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
GROUPSCOUT_COMPOSE="podman compose" make smoke-ui-docker-e2e
```

The current target name is Docker-specific. Do not rename it until source scripts and CI expectations are updated together. Without `GROUPSCOUT_COMPOSE`, the target keeps the Docker-known-good default.

Validated from Git Bash on 2026-07-01 because Windows `make` was not installed:

```sh
export PATH="$PATH:/c/Users/alvin/AppData/Local/Microsoft/WinGet/Packages/Docker.DockerCompose_Microsoft.Winget.Source_8wekyb3d8bbwe"
cd /c/Users/alvin/GolandProjects/groupscout
GROUPSCOUT_COMPOSE="podman compose" \
GROUPSCOUT_BACKEND_REPO=C:/Users/alvin/GolandProjects/groupscout \
GROUPSCOUT_UI_REPO=C:/Users/alvin/WebstormProjects/groupscout-ui \
GROUPSCOUT_UI_PRODUCTION_HOST_PORT=3004 \
GROUPSCOUT_UI_BAD_PROXY_HOST_PORT=3005 \
  bash scripts/smoke-ui-docker-e2e.sh
```

Ports `3004` and `3005` avoided a local rootless Podman relay issue where an earlier standalone `3002:3000` run bound only `[::1]:3002` and returned an empty host response. The production UI image itself responded correctly inside the container and on an explicit IPv4 host mapping.

Runtime-configurable backend Make targets:

```sh
make docker-up
make docker-down
make docker-logs
COMPOSE="podman compose" make docker-up
COMPOSE="podman compose" make docker-down
COMPOSE="podman compose" make docker-logs
```

The names stay Docker-specific for compatibility, but the `COMPOSE` variable allows Podman validation without duplicating every target.

## Known Migration Risks

### Promtail And Logs

The backend Compose file currently mounts Docker-specific paths for Promtail:

```yaml
/var/lib/docker/containers:/var/lib/docker/containers:ro
/var/run/docker.sock:/var/run/docker.sock:ro
```

That path is not portable to rootless Podman. Treat Loki/Promtail dashboard claims as unverified under Podman until the log source is changed to a Podman-compatible file path, journald source, or another collector strategy. Prometheus and Grafana can still start, but container labels and log labels may differ.

### Reverse Proxy Socket Discovery

Traefik examples in deployment docs use the Docker provider and `/var/run/docker.sock`. A Podman deployment must either expose a compatible Podman socket intentionally or use a static/file-provider reverse-proxy config. Do not mount a Docker socket path into Podman examples without validating the host socket and security posture.

### Host Aliases

Docker Desktop examples often use `host.docker.internal`. Podman commonly uses `host.containers.internal`, but behavior varies by OS, Podman machine, and rootless setup. Inside the backend/UI Compose network, prefer service DNS such as `http://groupscout:8080`, `http://postgres:5432`, and `http://ollama:11434`.

### Rootless Ports And Services

Rootless Podman can have different behavior for privileged host ports, user services, socket activation, DNS, and systemd-managed long-running services. GroupScout's default published ports are unprivileged (`3001`, `3002`, `5432`, `5678`, `8080`, `8081`, `9090`, `3100`), but production reverse-proxy work on `80` or `443` needs separate validation.

### GPU And Ollama

The current Ollama GPU notes are Docker-tested only. The Compose file includes NVIDIA device reservations and `NVIDIA_VISIBLE_DEVICES=all`; verify Podman GPU support separately before claiming local LLM enrichment is healthy under Podman. Basic API/UI smoke can pass even when `/health` reports `"ollama":"unavailable"`.

### Named Volumes

Docker and Podman keep separate local volume stores. Before switching runtimes on a machine with useful local data, export the Postgres and Ollama volumes from the old runtime or accept a fresh local stack.

**Verified 2026-07-02 — the n8n volume gap silently breaks Slack cadence.** After the switch, the Podman `groupscout_n8n_data` volume came up empty, so the `n8n` container had **no workflows** — the daily lead cadence never fired and no Slack push happened. This is separate from any container being down: even with n8n running, an empty volume means no active schedule trigger. Detect and recover:

```sh
podman exec groupscout_n8n n8n list:workflow          # empty output == workflow lost
podman cp backend/docs/workflows/n8n/sunday-wednesday-lead-cadence.json groupscout_n8n:/tmp/cadence.json
# In Git Bash prefix with MSYS_NO_PATHCONV=1, or the /tmp path is rewritten to a Windows path and import fails with ENOENT:
podman exec groupscout_n8n n8n import:workflow --input=/tmp/cadence.json
podman exec groupscout_n8n n8n publish:workflow --id=groupscout-sunday-wednesday-lead-cadence
podman restart groupscout_n8n                          # re-registers the schedule trigger
```

The cadence workflow is fully env-var driven (`GROUPSCOUT_OPS_SLACK_WEBHOOK_URL`, `GROUPSCOUT_API_TOKEN`, `GROUPSCOUT_API_BASE_URL` are set on the `n8n` service), so no n8n credentials need re-adding. See [N8N_GUIDE.md § 8 Troubleshooting](./N8N_GUIDE.md#8-troubleshooting--tips).

### Machine Autostart On Windows

The Podman machine does **not** start on Windows logon/boot. If it is down, every GroupScout container is down and the n8n schedule trigger never fires — the most common cause of a silently missed Slack cadence after a reboot. Check with `podman machine list`; `LAST UP: Never` or `Stopped` means nothing is running.

A Windows scheduled task, **`PodmanMachineAutostart`** (trigger: AtLogOn, current user), closes this gap. It runs `C:\Users\alvin\.local\share\containers\podman-autostart.ps1`, which:

1. runs `podman machine start` (no-op if already up),
2. polls `podman info` until the client connection is ready,
3. starts the stack in dependency order: `groupscout_postgres`, then `groupscout_ollama`, `groupscout_app`, `groupscout_n8n`.

The task owns container bring-up on purpose. Containers are `restart=unless-stopped`, but Podman's in-machine `podman-restart.service` **cannot be enabled over non-interactive `podman machine ssh`** (`Access denied`, a WSL/dbus limitation), so `unless-stopped` alone does not bring containers back after a machine restart. Verified end-to-end from a cold `podman machine stop`: the task brings the machine and all four containers up with the cadence workflow active.

Manage or re-run the task from PowerShell:

```powershell
Get-ScheduledTask -TaskName PodmanMachineAutostart | Select-Object TaskName, State
Start-ScheduledTask -TaskName PodmanMachineAutostart          # run it now
(Get-ScheduledTaskInfo -TaskName PodmanMachineAutostart).LastTaskResult   # 0 == success
```

Note: the trigger fires at user logon, not at the pre-login lock screen. Headless start before any user signs in would require a rootful machine plus a system service.

## Rollback

If Podman smoke fails and the Docker baseline is needed again:

```sh
cd /mnt/c/Users/alvin/GolandProjects/groupscout
docker compose -p groupscout down
docker compose -p groupscout up -d --build
docker compose -p groupscout ps
curl -i --max-time 5 http://localhost:8080/health
```

For UI standalone smoke:

```sh
cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui
docker build --target production -t groupscout-ui-production .
docker run --rm -p 3002:3000 \
  -e UI_API_PROXY_TARGET=http://host.docker.internal:8080 \
  groupscout-ui-production
```

Do not delete Docker or Podman volumes during rollback unless the data reset is intentional.

## Documentation Update Rules

- Keep historical Docker phase records intact unless they are active runbooks.
- For active docs, say "container runtime" when Docker and Podman both apply.
- Keep exact `docker ...` commands where they are known-good Docker gates.
- Add Podman variants only after the command is validated.
- Use `COMPOSE` and `CONTAINER_CLI` examples in new migration docs to avoid duplicating every command.
- Keep UI secrets server-side. `UI_SESSION_SECRET` is valid production server config, but it must not appear in browser-visible config, static assets, Compose output, or CI artifacts.
