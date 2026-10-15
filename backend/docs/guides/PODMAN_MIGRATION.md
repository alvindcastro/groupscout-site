# Podman Migration Runbook

This runbook captures the current Docker-to-Podman migration review for GroupScout. Docker remains the known-good runtime until the checks below are run against the backend and UI source repos; use this file to keep the migration explicit instead of scattering one-off command substitutions through older Docker docs.

Run backend runtime commands from `/mnt/c/Users/alvin/GolandProjects/groupscout` and UI runtime commands from `/mnt/c/Users/alvin/WebstormProjects/groupscout-ui`.

## Current Decision

> **Production host (IaC Phase 5, 2026-10-15):** production is moving from this Podman machine to a dedicated WSL 2 distro, `groupscout-prod`, which runs rootless Podman and is configured by Ansible. See [WSL_HOST.md](./WSL_HOST.md); the cutover is `groupscout-site-6ah.18`. After the cutover this machine is for development only. One finding from that work applies here too: all WSL distros share one cgroup tree and one `binfmt_misc` table (details in WSL_HOST.md).

- Podman is the runtime everywhere GroupScout runs:
  - This Podman machine runs development, and production until the cutover.
  - Production moves to rootless Podman on the WSL host (D4, 2026-10-15).
- The Compose files stay the source of truth. `podman compose` uses the standalone Docker Compose binary as its provider, and the GitHub Actions runners in CI use Docker.
- Docker-specific notes in older guides (DOCKER.md, HOME_DEPLOY.md) still apply to anyone running Docker, but Docker is not needed.
- 2026-07-01 verification: `RedHat.Podman` 5.8.3 and standalone `Docker.DockerCompose` 5.2.0 are installed. `podman machine init` and `podman machine start` provisioned a rootless WSL-backed machine, `podman --version`, `podman compose version`, and `podman info` passed, the backend stack started, the UI test image passed, and the backend-owned UI E2E smoke passed under Podman.
- Current Windows notes: PowerShell needs the Winget user PATH refreshed for `docker-compose.exe` after install. Git Bash can run the backend-owned smoke script with `GROUPSCOUT_COMPOSE="podman compose"` when the Docker Compose Winget package directory is on PATH. Plain WSL has `make`, and since 2026-10-14 it can also run the Windows `podman.exe` after the interop fix in [WSL Interop With systemd](#wsl-interop-with-systemd).
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

Since IaC Phase 1 (`groupscout-site-6ah.2`), the Compose file sets `name: groupscout`, so `-p groupscout` becomes optional but stays harmless. `make compose-check COMPOSE="podman compose"` renders every profile combination plus the production overlay. The layout and profiles are described in [DOCKER.md](./DOCKER.md#compose-layout).

3. Start the core backend stack and check health:

```sh
GROUPSCOUT_VERSION=$(git rev-parse --short HEAD) \
  $COMPOSE -p groupscout up -d --build postgres groupscout n8n
$COMPOSE -p groupscout ps
curl -i --max-time 5 http://localhost:8080/health
$COMPOSE -p groupscout exec postgres psql -U groupscout -d groupscout -c "SELECT 1;"
```

`/health` includes a `version` field with the SHA passed as `GROUPSCOUT_VERSION`. After IaC Phase 1, Ollama sits behind the `llm` profile. A targeted `up` like the one above no longer starts it as a dependency, so `/health` reports `"ollama":"degraded"`. Add `COMPOSE_PROFILES=llm` (or `llm,obs`) to `.env`, or prefix the command with it, when local LLM enrichment is needed.

4. Check logs without Docker-only container names:

```sh
$COMPOSE -p groupscout logs --tail=100 groupscout postgres n8n
COMPOSE_PROFILES=llm $COMPOSE -p groupscout logs --tail=100 ollama
```

5. Validate the UI test image:

```sh
cd /mnt/c/Users/alvin/WebstormProjects/groupscout-ui
podman build --target test -t groupscout-ui-test .
podman run --rm groupscout-ui-test
```

6. Validate merged backend plus UI Compose config. List the backend `docker-compose.override.yml` too. An explicit `-f` skips the auto-loaded dev override, and without it, `up` recreates the backend containers without their host ports (`8080`, `5432`, …).

```sh
COMPOSE="podman compose"
$COMPOSE -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.override.yml \
  -f compose.dev.yml \
  config --quiet
```

7. Smoke the development UI product server:

```sh
$COMPOSE -p groupscout \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.yml \
  -f /mnt/c/Users/alvin/GolandProjects/groupscout/docker-compose.override.yml \
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
COMPOSE="podman compose" make compose-check
```

The names stay Docker-specific for compatibility, but the `COMPOSE` variable allows Podman validation without duplicating every target. `make docker-up` exports `COMPOSE_PROFILES=llm,obs` and stamps the image with the git SHA. Pass `COMPOSE_PROFILES=` for the core services only.

## Known Migration Risks

### Logs (Alloy)

Promtail read `/var/lib/docker/containers`, which does not exist in the Podman machine, so the `obs` profile's log shipping never worked under Podman. IaC Phase 2 replaced it with Grafana Alloy, which reads logs through the Docker API at `/var/run/docker.sock`. Inside `podman-machine-default`, `/var/run/docker.sock` and `/run/docker.sock` are symlinks to the rootless `/run/user/1000/podman/podman.sock`, so the same mount works on both runtimes.

Verified 2026-10-14 on Podman 5.8.4: Alloy shipped every container's logs to Loki with `container`/`compose_service` labels, and Grafana's provisioned datasources and dashboard came up healthy. On first start, Alloy backfills each container's full log history, and Loki rejects anything older than one week. A burst of `timestamp too old` errors in the Alloy log right after the first start is expected.

### Log Rotation Options

The base Compose file sets `logging: json-file` with `max-size: 10m` and `max-file: 5` on every service. Podman 5.x maps `json-file` to its `k8s-file` driver and applies `max-size`. It passes `max-file` through as a generic option without erroring, but does not enforce it. That is acceptable for local development. Production hosts run Docker Engine, which honors both.

### Compose Profiles And Recovery Commands

After IaC Phase 1, `ollama`/`ollama-init` are in the `llm` profile and Prometheus/Grafana/Loki/Alloy are in the `obs` profile. Recovery commands that name services, such as `podman compose -p groupscout up -d postgres groupscout n8n`, start only the core services unless `COMPOSE_PROFILES` is set in `.env` or the shell. The `PodmanMachineAutostart` script (`deploy/windows/podman-autostart.ps1`) starts containers by name, including the `obs` set, so it is unaffected once the containers exist.

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

**Verified 2026-07-02 — the n8n volume gap silently breaks Slack cadence.** After the switch, the Podman `groupscout_n8n_data` volume came up empty, so the `n8n` container had **no workflows** — the daily lead cadence never fired and no Slack push happened. This is separate from any container being down: even with n8n running, an empty volume means no active schedule trigger.

**Fixed by IaC Phase 2 (2026-10-14).** The `n8n` service starts through `deploy/config/n8n/bootstrap.sh`, which imports and publishes every tracked workflow that is missing or changed before the server starts. An empty volume now recovers on the next `podman compose up -d n8n` or `podman restart groupscout_n8n`. This was verified against a fresh throwaway volume and on the live stack. Check it with:

```sh
podman exec groupscout_n8n n8n list:workflow --active=true
podman logs groupscout_n8n | grep groupscout-n8n-bootstrap
```

For a manual import outside the Compose stack, see the [workflow runbook](../workflows/n8n/README.md#importing-outside-the-compose-stack).

The cadence workflow is fully env-var driven (`GROUPSCOUT_OPS_SLACK_WEBHOOK_URL`, `GROUPSCOUT_API_TOKEN`, `GROUPSCOUT_API_BASE_URL` are set on the `n8n` service), so no n8n credentials need re-adding. See [N8N_GUIDE.md § 8 Troubleshooting](./N8N_GUIDE.md#8-troubleshooting--tips).

### WSL Interop With systemd

The Ubuntu WSL distro runs with `systemd=true`. At boot, systemd mounts a fresh, empty `binfmt_misc` table, which drops the `WSLInterop` entry. Every Windows binary (`cmd.exe`, `podman.exe`) then fails with `cannot execute binary file: Exec format error`. WSL generates `/run/systemd/generator/systemd-binfmt.service.d/override.conf` to re-register `WSLInterop` via `systemd-binfmt.service`, but two drop-ins skipped that unit on WSL:

- `/usr/lib/systemd/system/systemd-binfmt.service.d/wsl.conf` (`ConditionVirtualization=!wsl`)
- `/etc/systemd/system/systemd-binfmt.service.d/00-wsl.conf` (`ConditionVirtualization=!container`; `systemd-detect-virt --container` reports `wsl`)

Fixed 2026-10-14 with a drop-in that sorts last and resets those conditions:

```sh
# /etc/systemd/system/systemd-binfmt.service.d/zz-run-on-wsl.conf
[Unit]
ConditionVirtualization=
ConditionPathIsMountPoint=/proc/sys/fs/binfmt_misc
```

Then run `sudo systemctl daemon-reload && sudo systemctl start systemd-binfmt`. `ls /proc/sys/fs/binfmt_misc/` should list `WSLInterop`. The unit is wanted by `sysinit.target`, so the fix re-applies on every boot. To undo, delete the file and reload.

`binfmt_misc` tables are per distro. Changing `binfmt.d` inside `podman-machine-default` does not affect Ubuntu, and restarting `systemd-binfmt` inside the machine can stall its SSH logins for minutes.

Without interop, WSL can still drive Podman by forwarding the machine's rootless API socket over SSH. Take the port and key from `C:\Users\alvin\.config\containers\podman\machine\wsl\podman-machine-default.json`; copy the key with `install -m 600` first, because DrvFs shows it as `0777`:

```sh
ssh -i <key-copy> -p <SSH.Port> -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  -o StreamLocalBindUnlink=yes -N \
  -L /run/user/1000/podman-wsl.sock:/run/user/1000/podman/podman.sock user@127.0.0.1 &
export DOCKER_HOST=unix:///run/user/1000/podman-wsl.sock   # docker-compose / curl now reach Podman
```

Published container ports are reachable from WSL at `localhost` either way, because WSL2 distros share one network namespace.

### Machine Autostart On Windows

The Podman machine does **not** start on Windows logon/boot. If it is down, every GroupScout container is down and the n8n schedule trigger never fires — the most common cause of a silently missed Slack cadence after a reboot. Check with `podman machine list`; `LAST UP: Never` or `Stopped` means nothing is running. Since IaC Phase 9 a missed cadence also raises the Healthchecks.io alert ([MONITORING.md](./MONITORING.md)).

A Windows scheduled task, **`PodmanMachineAutostart`** (trigger: AtLogOn, current user), closes this gap. Since IaC Phase 10 (2026-10-15) it is code in the backend repo, `deploy/windows/`:

| File | What it does |
|---|---|
| `podman-autostart.ps1` | Starts the machine, waits for `podman info`, starts `groupscout_postgres` and waits until it reports healthy, then starts `groupscout_ollama`, `groupscout_app`, `groupscout_n8n` and the `obs` containers (`groupscout_prometheus`, `groupscout_loki`, `groupscout_alloy`, `groupscout_grafana`). A container that doesn't exist yet is logged and skipped. Exit code 0 = all started, 1 = Podman never answered, 2 = a container failed to start. Log: `%LOCALAPPDATA%\GroupScout\podman-autostart.log` (rotated at 1 MB). |
| `register-autostart.ps1` | Copies the script to `%LOCALAPPDATA%\GroupScout` and creates or updates the task (no admin needed). The task runs that copy, so it never depends on a checkout or branch. Re-run it after changing the script. `-RunNow` runs the task and reports its result, `-Unregister` removes it, `-WhatIf` shows the plan. |
| `configuration.dsc.yaml` | `winget configure` file for a new workstation: WSL, Podman, Docker Compose, Git, GitHub CLI, Go, Node LTS, age, sops. |
| `setup-dev.ps1` | Second step on a new workstation: creates and starts the Podman machine if needed, checks the Compose provider, creates `.env` from `.env.example` if missing (never writes secrets), registers the task, then `podman compose up -d --build` with `llm,obs` and waits for `/health`. `-SkipStack`, `-SkipAutostart`, `-WhatIf`. |

The task owns container bring-up on purpose. Containers are `restart=unless-stopped`, but Podman's in-machine `podman-restart.service` **cannot be enabled over non-interactive `podman machine ssh`** (`Access denied`, a WSL/dbus limitation), so `unless-stopped` alone does not bring containers back after a machine restart. `groupscout_alertd` is not in the list: it has not run under Podman since the migration (`groupscout-site-q74`).

Install or update the task from PowerShell in the backend checkout:

```powershell
powershell -ExecutionPolicy Bypass -File deploy\windows\register-autostart.ps1 -RunNow
Get-Content $env:LOCALAPPDATA\GroupScout\podman-autostart.log -Tail 15
(Get-ScheduledTaskInfo -TaskName PodmanMachineAutostart).LastTaskResult   # 0 == success
```

**New workstation, two commands** (from a checkout of the backend repo; without Git, download the repo zip from GitHub):

```powershell
winget configure -f deploy\windows\configuration.dsc.yaml --accept-configuration-agreements
powershell -ExecutionPolicy Bypass -File deploy\windows\setup-dev.ps1
```

The first command asks for elevation for WSL and may need a reboot before Podman can create its machine; re-running either command is safe. The stack starts with placeholder values from `.env.example`; render the real ones with `make secrets-render` in WSL ([SECRETS.md](./SECRETS.md)). `winget configure test -f deploy\windows\configuration.dsc.yaml --accept-configuration-agreements` reports drift without changing anything.

History: until 2026-10-15 the task ran a hand-made `C:\Users\alvin\.local\share\containers\podman-autostart.ps1` that started only Postgres, Ollama, the app and n8n, with no log. That script and the previous task definition are kept in `C:\Users\alvin\groupscout-backups\2026-10-15-iac-phase10\` (restore with `Register-ScheduledTask -TaskName PodmanMachineAutostart -Xml (Get-Content -Raw <xml>) -Force`).

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
