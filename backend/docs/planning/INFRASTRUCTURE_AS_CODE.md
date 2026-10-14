# Infrastructure as Code: Brainstorm & Task Plan

Date: 2026-10-14
Status: Phase 1 implemented on task branches (awaiting runtime verification and merge). Other phases planned.

> **Tracking:** Beads is the source of truth. Epic `groupscout-site-6ah` has one child issue per phase, linked by the dependencies in §8. Phase N is `groupscout-site-6ah.<N+1>`, and each phase heading below shows its ID. The task lists in this doc describe scope only; check status with `bd show <id>`. The existing home deploy, backups, and restore-smoke work is `groupscout-site-39g`, linked as related to Phase 8.

Related docs: [DEPLOYMENT_OPTIONS.md](./DEPLOYMENT_OPTIONS.md) · [HOME_DEPLOY.md](../guides/HOME_DEPLOY.md) · [COOLIFY.md](../guides/COOLIFY.md) · [PODMAN_MIGRATION.md](../guides/PODMAN_MIGRATION.md) · [DOCKER.md](../guides/DOCKER.md) · [N8N_GUIDE.md](../guides/N8N_GUIDE.md) · [n8n workflows](../workflows/n8n/README.md)

---

## 1. Why IaC Now

Most recent operational incidents share one root cause: the running system drifted from what the repos describe, and nothing noticed.

| Incident (from history and runbooks) | Root cause | What IaC changes |
|---|---|---|
| Dockerfile built only `cmd/server/main.go`. No image built successfully for weeks, so merged fixes never reached the container. | No automated image build on merge | CI builds the image on every PR and merge. A broken build fails the PR instead of failing silently. |
| Closed issues referenced fixes that existed only on unmerged branches. The live image predated them. | No link between a running container and a commit | Images are tagged with the git SHA. `/health` reports the build SHA. Deploys use a pinned tag from `main` only. |
| The Podman switch left `groupscout_n8n_data` empty. The lead cadence silently stopped and no Slack pushes went out. | The workflow existed only inside a volume | An `n8n-init` step imports workflows from git on every boot. A dead-man's-switch check alerts when the cadence doesn't run. |
| The Podman machine didn't autostart on Windows, so all containers were down. | Host setup was done by hand | The autostart script and scheduled-task registration live in the repo and are applied by one command. |
| `podman-autostart.ps1` exists only at `C:\Users\alvin\.local\share\containers\`. | Host config isn't versioned | Moved into the repo under `deploy/windows/`. |
| Live n8n schedule edits were made through export, edit, and import. | Click-ops on a live system | The JSON in git is the source of truth. Deploy re-imports it. |
| Every image uses `:latest`, and the n8n CLI changed between versions (`update:workflow --active` vs `publish:workflow`). | Unpinned images | Images are pinned by version and digest, and Renovate opens PRs for upgrades. |
| The home deploy, backup, and restore smoke (`39g`) still hasn't been executed. | A 425-line manual runbook | An Ansible playbook, a scheduled backup timer, and a scripted restore smoke. |

---

## 2. Current-State Inventory

Observed 2026-10-14 in the backend repo (`GolandProjects/groupscout`) and the UI repo (`WebstormProjects/groupscout-ui`).

| Component | Where it's defined | Gaps found |
|---|---|---|
| Service topology (10 services) | Backend `docker-compose.yml` | Every port is published on `0.0.0.0` (5432, 5678, 8080, 8081, 9090, 3000, 3100). The Postgres password is hard-coded as `groupscout`. Most images use `:latest`. There are no profiles, no top-level `name:`, and no log rotation. |
| Backend image | Backend `Dockerfile` | `alpine:latest` runtime is unpinned (already flagged in NOT_DONE_AND_UPGRADES). No version or SHA is baked in. |
| UI image | UI `Dockerfile` + `compose.dev.yml` | Built from a hard-coded local path (`/mnt/c/Users/...`). No registry image. The production stage inherits the test stage, so test files ship in the image. |
| Prometheus | Compose only | **No scrape config mounted.** The app exposes `/metrics` (`cmd/server/main.go:127`), but nothing scrapes it. |
| Grafana | Compose only | No provisioned datasources or dashboards. Default admin credentials. |
| Logs | Promtail + `config/promtail.yaml` | Mounts the Docker socket (root-equivalent) and Docker log paths, so it isn't Podman-compatible. Promtail is deprecated in favour of Grafana Alloy. |
| Ollama | Compose | `mem_limit: 12g` plus an NVIDIA GPU reservation. This **won't fit** the recommended Hetzner CX32 (8 GB, no GPU). |
| n8n | Compose + workflow JSON in the **coordination repo** | Imported by hand. `N8N_ENCRYPTION_KEY` isn't set, so it's auto-generated inside the volume. The image is unpinned. |
| Secrets | `.env` on disk | Plaintext and unversioned. `alertd` uses `env_file: .env`, so it receives every secret. |
| CI/CD | None | No `.github/` in either repo. |
| Host (current) | Windows + WSL2 + Podman machine | Autostart scheduled task and `.ps1` are outside version control. Containers start only at user logon. |
| Backups | `scripts/backup-volumes.sh` | Docker-only. Writes to local disk only, with no offsite copy, no schedule, and no restore test. |
| Cloud IaC | PHASES.md / PROMPTS_PHASE25 Part D (GCP Terraform) | Never started, and the target is wrong: Cloud Run can't host `alertd`. Should be marked superseded. |

---

## 3. Goals and Non-Goals

**Goals**

- Rebuild the production host from zero in **under 1 hour** with one command plus the secrets key.
- Trace every running container to a git commit (image tag = SHA, `/health` shows it).
- Make it impossible for the lead cadence to stop silently (workflow-as-code plus an external heartbeat).
- Keep secrets encrypted at rest in git, so nothing lives only in a local `.env`.
- Make drift detectable, so a scheduled plan/check reports differences.
- Use one IaC codebase for both candidate targets: a home Linux box and a Hetzner VPS.

**Non-goals**

- Kubernetes, Nomad, or any orchestrator. One host is enough for an internal tool.
- High-availability Postgres, multi-region, or autoscaling.
- GCP Cloud Run. `alertd` is a polling daemon and can't scale to zero.
- Making the Windows dev workstation a production host.

---

## 4. Layered Model

```
┌──────────────────────────────────────────────────────────────────────┐
│ L5  Monitoring as code   uptime checks, cadence heartbeat, alert rules│
│ L4  Delivery             GitHub Actions → GHCR images → deploy job     │
│ L3  App config           n8n workflows, Prometheus/Grafana/Alloy cfg  │
│ L2  Workload             Compose files + profiles, pinned images      │
│ L1  Host config          Ansible: OS, firewall, runtime, timers       │
│ L0  Resources            OpenTofu: VPS, DNS, tunnel, buckets, monitors│
├──────────────────────────────────────────────────────────────────────┤
│ ⟂   Secrets plane        SOPS + age, used by L0–L4                    │
└──────────────────────────────────────────────────────────────────────┘
```

L2 and L3 deliver value **on any target, including today's Windows/Podman setup**, so start there. L0 and L1 depend on the deployment-target decision (D1).

---

## 5. Key Decisions

Each decision lists the options, a recommendation, and the reason. Phase 0 (`groupscout-site-6ah.1`) records the final outcome of each one here, including any change from the recommendation.

### D1: Deployment target

- (a) Stay on the Windows workstation with the Podman machine. Uptime depends on a desktop and on user logon.
- (b) A home Linux box with Cloudflare Tunnel. $0, and the current `39g` path.
- (c) A Hetzner VPS. About $10/month and the best uptime.

**Recommendation:** Write L1–L5 against an abstract target, "one Debian/Ubuntu host reachable over SSH." Targets (b) and (c) then share 100% of the code, and only the Ansible inventory and the L0 module differ. Pick the first real target separately. Keep (a) for development only.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D2: Provisioning tool (L0)

- OpenTofu, Terraform, Pulumi (Go), or none.

**Recommendation:** **OpenTofu.** It's MPL-licensed, uses the same `hcloud` and `cloudflare` providers, and has **native client-side state encryption**. That matters here because state will hold tunnel tokens. Pulumi in Go is a reasonable alternative if you prefer writing Go over HCL. "None" is valid for a home-only target, where L0 shrinks to Cloudflare resources plus a backup bucket.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D3: Host configuration (L1)

- Ansible, cloud-init only, or NixOS.

**Recommendation:** **Ansible.** Runs are idempotent, `--check --diff` gives you drift detection, and it works on a home box that has no cloud-init. On Hetzner, cloud-init only creates the deploy user and SSH key, and Ansible does the rest. NixOS is powerful, but it's a big learning step for one host.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D4: Container runtime on the server

- Docker Engine + Compose, or Podman + Quadlets (systemd units).

**Recommendation:** **Docker Engine + Compose for v1.** It's the documented production baseline, and existing Traefik, Promtail, and smoke scripts assume it. On Linux, the Docker daemon starts at boot, so `restart: unless-stopped` works there, unlike on the Podman machine. Revisit Quadlets later; `podlet` can convert the Compose file.
**Gotcha:** Docker-published ports **bypass UFW**. In production, bind ports to `127.0.0.1` or don't publish them at all.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D5: Ingress

- Cloudflare Tunnel, Traefik + Let's Encrypt + port forwarding, or Tailscale only.

**Recommendation:** **Cloudflare Tunnel** with the tunnel, DNS, and Access policies defined in OpenTofu. This means no open inbound ports on either target and no CGNAT problems. Put **Cloudflare Access** (free for small teams) in front of n8n, Grafana, the UI, and alertd. Expose only the API hostname publicly, protected by its bearer token. Use **Tailscale** for SSH and admin access.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D6: Coolify

**Recommendation:** **Drop it on the IaC path.** Coolify keeps its config in its own database and UI, which conflicts with git being the source of truth, and it has a recent CVE history. Keep COOLIFY.md as the click-ops alternative.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D7: Secrets

- SOPS + age, Ansible Vault, 1Password or Bitwarden Secrets Manager, or Infisical/Doppler.

**Recommendation:** **SOPS + age.** It's free and git-native, and it integrates with Ansible (`community.sops`) and OpenTofu (sops provider). It encrypts values, not whole files, so diffs stay readable. The age private key is the root of trust: store it in a password manager and keep one offline backup.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D8: Where the IaC code lives

- (a) A `deploy/` folder in the backend repo, or (b) a new `groupscout-infra` repo.

**Recommendation:** **(a) to start.** Compose already lives in the backend repo, so one PR can change the app and its infrastructure together, and one CI setup covers both. The UI repo publishes an image to GHCR that the backend's `deploy/` consumes by tag. Long-lived docs stay in this coordination repo, following the existing convention. Split into a separate repo only if a second app or a different deploy cadence appears.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D9: OpenTofu state backend

**Recommendation:** An S3-compatible bucket (Cloudflare R2, Hetzner Object Storage, or Backblaze B2) with OpenTofu state encryption turned on. The state bucket is created once by hand, or by a tiny `bootstrap/` stack with local state, to solve the chicken-and-egg problem. Document that step.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D10: Ollama in production

- Disable it on the VPS (Claude-only enrichment), buy a bigger or GPU box, or keep Ollama on a home GPU machine and reach it over Tailscale.

**Recommendation:** Move Ollama into an `llm` Compose profile that's **off by default in production**, and set `OLLAMA_ENABLED=false` there. Revisit after measuring how often the Ollama paths actually matter. Hetzner GPU servers cost far more than the rest of the stack combined.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### D11: Scheduling

- Keep n8n with workflow-as-code, or move the cadence to a systemd timer or the in-app scheduler.

**Recommendation:** Keep n8n for now and add `n8n-init` (Phase 2). Moving the cadence would be a product change, not an infrastructure change. Track it separately if it's wanted.

**Outcome:** pending (Phase 0, `groupscout-site-6ah.1`).

### Recommended stack at a glance

| Layer | Tool |
|---|---|
| L0 Resources | OpenTofu (`hcloud`, `cloudflare`, S3 backend with state encryption) |
| L1 Host | Ansible + cloud-init bootstrap |
| L2 Workload | Docker Engine + Compose (base + prod override + profiles) |
| L3 App config | Files in git, mounted read-only, plus an `n8n-init` import step |
| L4 Delivery | GitHub Actions → GHCR → SSH over Tailscale, with environment approval |
| L5 Monitoring | Healthchecks.io-style heartbeat, external uptime check, Grafana alert rules |
| Secrets | SOPS + age |
| Upgrades | Renovate (image digests, Actions SHAs, providers) |

---

## 6. Proposed Layout (backend repo)

```
groupscout/
├── docker-compose.yml              # base: local build, pinned images, healthchecks, profiles, no host ports  ✅ Phase 1
├── docker-compose.override.yml     # dev host ports, auto-loaded when no -f is given  ✅ Phase 1
├── deploy/
│   ├── compose/
│   │   └── compose.prod.yml        # prod overlay: registry image by tag, required secrets, 127.0.0.1 binds  ✅ Phase 1
│   ├── config/
│   │   ├── prometheus/prometheus.yml
│   │   ├── grafana/provisioning/{datasources,dashboards,alerting}/
│   │   ├── alloy/config.alloy
│   │   └── n8n/workflows/*.json    # runtime artifact, synced from coordination repo
│   ├── secrets/
│   │   └── prod.env.sops           # SOPS-encrypted dotenv
│   ├── ansible/
│   │   ├── inventory/{home.yml,hetzner.yml}
│   │   ├── group_vars/all/{main.yml,secrets.sops.yml}
│   │   ├── roles/{base,hardening,docker,tailscale,groupscout,backups,observability}/
│   │   └── site.yml
│   ├── tofu/
│   │   ├── bootstrap/              # state bucket (local state)
│   │   ├── modules/{hetzner_host,cloudflare_edge,backup_bucket,monitors}/
│   │   └── envs/prod/
│   ├── scripts/{deploy.sh,verify.sh,backup.sh,restore-smoke.sh}
│   └── windows/
│       ├── configuration.dsc.yaml  # winget configure: podman, compose, git, go, node
│       ├── podman-autostart.ps1
│       └── register-autostart.ps1
├── .sops.yaml
├── renovate.json
└── .github/workflows/{ci.yml,images.yml,infra.yml,deploy.yml,drift.yml}
```

---

## 7. Phased Task Plan

Sizes: **S** is about one session, **M** is a few sessions, and **L** is a week or more of evenings. Phases 1–4 don't depend on D1. The bullets are scope; Beads tracks status.

### Phase 0: Decisions and groundwork (S) · `groupscout-site-6ah.1`

- Review D1–D11 and record the outcome of each in §5.
- ~~Mark PHASES.md / PROMPTS_PHASE25 Part D (GCP Terraform) as superseded~~ Done 2026-10-14, along with FUTURE_INTEGRATION.md and the ROADMAP Part D entry.
- ~~Add pointers from DEPLOYMENT_OPTIONS.md, ROADMAP, NOT_DONE_AND_UPGRADES, WHERE_TO_START and the root README~~ Done 2026-10-14.
- ~~Create the Beads epic and per-phase issues~~ Done 2026-10-14: `groupscout-site-6ah` with `.1`–`.12`. `39g` is linked as related to Phase 8, so it stays ready.

**Done when:** the decisions are recorded.

### Phase 1: Harden the existing Compose and images (M), no new tools · `groupscout-site-6ah.2`

Implemented 2026-10-14 on backend branch `task/iac-phase1-hardening` and UI branch `task/iac-ui-production-image`. Neither is merged yet.

Compose (backend repo):
- Top-level `name: groupscout`, so volume and network names match the existing `groupscout_*` resources under Docker and Podman alike.
- Three-file layout using Compose's built-in override convention instead of the `deploy/compose/compose.{base,dev}.yml` split first proposed in §6. Plain `docker compose up` and `podman compose up` keep working with no `-f` flags:
  - `docker-compose.yml` is the base: a local build stamped with the git SHA, and no host ports. The build stays in the base because the UI repo's `-f docker-compose.yml -f compose.dev.yml` flows skip the override. A first version put the build in the override, which left those flows with nothing to build (fixed in `71eb547`).
  - `docker-compose.override.yml` is dev: today's host ports only. Multi-file `-f` commands in the docs now list it explicitly, because without it `up` recreates the backend without host ports.
  - `deploy/compose/compose.prod.yml` is prod: `GROUPSCOUT_IMAGE`/`GROUPSCOUT_IMAGE_TAG` required, `build: !reset null` so hosts never build, secrets required through `${VAR:?}`, ports bound to `127.0.0.1` only, memory limits, `JSON_LOG=true`, and Ollama off by default.
- Every image is pinned to the version `:latest` resolved to on 2026-10-14, so pinning never downgrades a volume:
  - n8n `2.43.5`
  - Grafana `13.2.4`
  - Prometheus `v3.15.0`
  - Loki `3.7.8`
  - Promtail `3.6.11`
  - Ollama `0.40.4`
  - pgvector `0.8.7-pg17-bookworm`
- pgvector pins the `-bookworm` variant explicitly. A silent move of the `pg17` tag to a Debian trixie base would change glibc collation under the existing data directory.
- Secrets come from env. Dev keeps working defaults, and prod requires real values:
  - `POSTGRES_PASSWORD`, which also builds `DATABASE_URL`
  - `N8N_ENCRYPTION_KEY`. A blank value keeps the key stored in the volume; n8n only compares keys when the variable is non-empty.
  - `GRAFANA_ADMIN_PASSWORD`
- `GENERIC_TIMEZONE`/`TZ` are set to `America/Vancouver`. The cadence workflow already pins that timezone, so its schedule is unchanged.
- `alertd` drops `env_file: .env` and receives only the 6 variables `cmd/alertd` reads. This also lets `compose config` run without a `.env` (CI, fresh clones).
- Healthchecks on `groupscout` (`/health`), `alertd` (new `/health`) and `n8n` (`/healthz`), plus json-file log rotation on every service.
- Profiles: core services have none, `llm` covers Ollama and ollama-init, and `obs` covers Prometheus, Grafana, Loki and Promtail. groupscout's Ollama dependencies use `required: false`. The `ui` profile stays in the UI repo's `compose.dev.yml` rather than being duplicated here.
- Removed the unused `SENDGRID_API_KEY` passthrough.
- `make compose-check` renders every dev profile set plus the prod overlay. `internal/smoke/compose_hardening_test.go` runs under `go test ./...` and fails on any of these regressions, with no container runtime needed:
  - a `:latest` image
  - host ports in the base file
  - non-loopback prod ports
  - a missing core healthcheck or log rotation
  - `alertd` using an `env_file`
  - an unpinned or root Dockerfile

Images and build:
- `/health` returns `version`, stamped with `-ldflags "-X main.version=<sha>"`. The handler was extracted into `cmd/server/health.go` with unit tests.
- Dockerfile:
  - pinned `golang:1.26.9-alpine3.24` and `alpine:3.24.2`
  - OCI `title`/`source`/`version`/`revision` labels. `created` is left to the Phase 4 CI metadata step.
  - runs as uid 10001
  - builds `./cmd/alertd` as a package. It previously compiled the single file `main.go`, the same bug that broke the server image.
- New `.dockerignore`, so `.env` and local binaries such as the 33 MB `server.exe` stay out of the build context.
- Makefile:
  - `COMPOSE` and `COMPOSE_PROFILES` (default `llm,obs`) variables
  - `docker-up` runs with `--build`, so the running image matches the checkout
  - fixed `run`, `run-once`, `ollama-push` and `start-fresh`, which ran `go run cmd/server/main.go` and no longer compiled
- UI: the production stage now starts from a clean `node:22.23.3-bookworm-slim` base, copies only `package.json` and `web/`, and runs as `node`. The D4 contract test was updated to match.

Verified in WSL without a container runtime:
- `go vet ./...` and `go test ./...` pass.
- The eval gate passes.
- Compose v5.6.0 `config` renders all four dev profile sets plus prod, and prod rejects missing secrets.
- The UI suite has 146 passing, 0 failing and 1 skipped. The pre-existing D5 docs failure (`groupscout-site-7t7`) was a stale regex, fixed on the UI branch.
- The coordination docs describing the new layout were updated the same day (see the backend CHANGELOG entry for 2026-10-14). Their "lands with `task/iac-phase1-hardening`" notes come out when this phase closes.

**Still to verify on Podman before closing:**
- `make compose-check COMPOSE="podman compose"`
- `make docker-up`, then check that `/health` shows the SHA and `podman ps` lists the core services as healthy
- the UI E2E smoke
- the cadence workflow still active after the n8n upgrade

### Phase 2: App config as code (M) · `groupscout-site-6ah.3`

n8n:
- Copy the workflow JSON to `deploy/config/n8n/workflows/`. Decide whether the coordination repo copy becomes the doc reference or a sync target, and remove the duplication risk.
- Add an `n8n-init` service (same pinned image and volume) that runs **before** `n8n` (`depends_on: service_completed_successfully`):
  - Import each workflow **only if it's missing or its checksum changed**, to preserve the `staticData` cadence-dedupe keys.
  - Activate or publish the workflow using the command that matches the pinned n8n version.
  - Exit non-zero if import or activation fails, so `n8n` doesn't start in a broken state.
- Remove the manual "import, activate, restart" steps from PODMAN_MIGRATION.md and N8N_GUIDE.md once `n8n-init` is verified.
- Test: start with an empty `n8n_data` volume and confirm the cadence workflow is active with no manual steps. This reproduces the Podman incident.

Observability:
- Add `deploy/config/prometheus/prometheus.yml` that scrapes `groupscout:8080/metrics`, plus `alertd` if it exposes metrics.
- Add Grafana provisioning: Prometheus and Loki datasources, plus a starter "GroupScout overview" dashboard JSON (run counts, lead counts, collector errors, HTTP latency).
- Replace Promtail with **Grafana Alloy**, using a config that works for Docker (socket) and is documented for Podman (journald or file).
- Mount all config read-only, so nothing important lives only in the `grafana_data` or `prometheus_data` volumes.

**Done when:** `down -v` followed by `up` brings back the active workflow, the scrape targets, and the dashboards with zero manual steps.

### Phase 3: Secrets as code (S) · `groupscout-site-6ah.4`

- Generate an age keypair. Store the private key in a password manager plus one offline backup, and document the recovery steps.
- Add `.sops.yaml` with creation rules for `deploy/secrets/*.sops*` and `ansible/**/secrets.sops.yml`.
- Build `deploy/secrets/prod.env.sops` from the inventory in Appendix A. Don't commit the plaintext `.env`.
- Add `make secrets-edit` (`sops deploy/secrets/prod.env.sops`) and `make secrets-render`, which decrypts to a `0600` tmpfs or ignored path.
- Add a pre-commit hook or CI check that fails if anything matching `*.env` (not `.sops`), `*.pem`, or a token pattern gets committed (e.g. gitleaks).
- Rotate secrets that were ever shared in plaintext or are defaults: the Postgres password, the Grafana admin password, and `API_TOKEN`.

**Done when:** a fresh clone plus the age key can render a working `.env`, and the secret scanner runs in CI.

### Phase 4: CI with build, scan, and publish (M) · `groupscout-site-6ah.5`

- `ci.yml` (backend): `go vet`, `go test ./...`, `make eval-gate`, `docker build`, and `docker compose config` for each profile, on every PR.
- `ci.yml` (UI): `npm test` plus the production-target image build on every PR.
- `images.yml`: on merge to `main`, build and push `ghcr.io/<owner>/groupscout:<sha>` and `groupscout-ui:<sha>`, plus a moving `main` tag.
- Image vulnerability scan (Trivy or Grype) that fails on fixable critical findings.
- Pin every third-party Action to a commit SHA.
- Add `renovate.json` covering Docker digests in Compose and Dockerfiles, GitHub Actions SHAs, OpenTofu providers, Ansible collections, and Go modules (grouped and scheduled weekly).
- Branch protection on `main`: CI must pass before merging.
- Add a backend-owned integration-test job with a Postgres service container, keeping the advisory-lock serialization that already exists.

**Done when:** a broken Dockerfile can't merge, and every `main` commit has a pullable image tagged with its SHA.

### Phase 5: Host configuration with Ansible (L), needs D1/D3/D4 · `groupscout-site-6ah.6`

- Write the inventory for the chosen target(s) and `group_vars` (domain, timezone, deploy user, backup bucket).
- Role `base`: packages, timezone, `unattended-upgrades`, chrony/NTP, swap file sized for 8 GB hosts.
- Role `hardening`: deploy user with an SSH key only, no root login, no password auth, UFW default-deny inbound, SSH allowed only on the Tailscale interface.
- Role `tailscale`: install and join with an auth key from SOPS, and tag the node `tag:groupscout`.
- Role `docker`: install Docker Engine from the official repo, `daemon.json` (log rotation, `live-restore`), and add the deploy user to the `docker` group.
- Role `groupscout`: create `/opt/groupscout`, render the Compose files and decrypted `.env` (`0600`), log in to GHCR with a read-only token, then `docker compose up -d` with the prod and profile flags.
- Role `backups`: install restic and the systemd units from Phase 8.
- Make `site.yml` idempotent: a second run reports `changed=0`.
- Use `ansible-lint` in CI, and run `ansible-playbook --check --diff` as the drift check.

**Done when:** one command takes a fresh Debian/Ubuntu host to a running, healthy stack, and a re-run makes no changes.

### Phase 6: Cloud resources with OpenTofu (M), needs D1/D2/D5/D9 · `groupscout-site-6ah.7`

- `bootstrap/`: state bucket and a scoped API key, using local state (commit only the encrypted state or none at all).
- Configure `envs/prod` backend and state encryption (passphrase or key from SOPS).
- Module `cloudflare_edge`: tunnel, tunnel ingress config (api, ui, n8n, grafana hostnames), DNS records, and Access application + policy (allow-list emails) for every hostname except the API. Output the tunnel token into SOPS or an Ansible var.
- Module `hetzner_host` (only if D1 = c): server (CX32, Ubuntu LTS), firewall that blocks all inbound since the tunnel and Tailscale are outbound, SSH key, cloud-init that creates the deploy user, and a daily-backup flag.
- Module `backup_bucket`: bucket with a lifecycle rule and a write-only key for restic.
- Module `monitors` (L5): uptime check on the public `/health`, plus the cadence heartbeat check (Phase 9).
- Use `tofu fmt -check`, `tofu validate`, and `tflint` in CI, and post `tofu plan` as a PR comment.

**Done when:** `tofu apply` creates the edge and host resources from scratch, and `tofu plan` on an unchanged tree shows no changes.

### Phase 7: Deploy pipeline (M) · `groupscout-site-6ah.8`

- `deploy/scripts/deploy.sh <sha>`: take a pre-deploy `pg_dump`, set the image tag, `docker compose pull`, `docker compose up -d --remove-orphans`, run `verify.sh`, and roll back automatically to the previous tag if the check fails.
- `verify.sh`: `/health` returns 200 with `database: ok` and the expected SHA, `alertd` is healthy, the n8n workflow is active, and the UI `/healthz` returns 200.
- `deploy.yml`: manually triggered (or triggered on a release tag), with GitHub Environment `production` requiring approval. The runner joins Tailscale with an ephemeral key, then SSHes in and runs `deploy.sh <sha>`.
- Record each deploy (SHA, time, actor) in a `deploys.log` on the host or as a GitHub deployment.
- Document rollback: `deploy.sh <previous-sha>`.
- Confirm the app runs DB migrations on startup safely, and document the expand/contract rule for destructive migrations.

**Done when:** a merge to `main` followed by one approved click puts a verified build in production, and a bad build rolls itself back.

### Phase 8: Backups and disaster recovery (M), closes the backup part of `39g` · `groupscout-site-6ah.9`

- Write `backup.sh`: `pg_dump -Fc` plus the n8n volume (and Grafana, if anything there isn't provisioned) into restic, sent to the offsite bucket.
- Add a systemd timer that runs nightly, with retention of 7 daily, 4 weekly, and 6 monthly.
- Have the backup ping a heartbeat check when it succeeds, so a failed or missed backup raises an alert.
- Write `restore-smoke.sh`: restore the latest snapshot into a throwaway Postgres container, then run sanity queries (row counts on `leads` and `raw_inputs`, and confirm the `vector` extension is present).
- Run the restore smoke monthly with a timer and once by hand now, recording the result in `39g`.
- Write a DR runbook (in this repo) covering a total host loss: new host, `tofu apply`, `ansible-playbook site.yml`, restore, verify. Time the drill against the 1-hour goal.
- Retire or rewrite `scripts/backup-volumes.sh` (which is Docker-only and local-only) in favour of the above.

**Done when:** a timed DR drill rebuilds production from nothing in under an hour.

### Phase 9: Monitoring as code (S) · `groupscout-site-6ah.10`

- **Cadence dead-man's switch:** the n8n workflow (or the backend `/run` path) pings a heartbeat URL after each successful cadence run. The check expects a ping on Sun, Tue, and Thu by about 18:30 Pacific and alerts if one doesn't arrive. This is the check that would have caught the Podman/n8n silent failure.
- External uptime check on the public `/health`, alerting through email or Slack.
- Grafana alert rules, provisioned as code: collector error-rate spike, no leads in N days, Postgres disk above 80%, a container restart loop.
- Route alerts to a Slack channel separate from the lead notifications.

**Done when:** stopping the n8n container through a cadence window produces an alert without anyone checking manually.

### Phase 10: Dev workstation as code (S), Windows/Podman · `groupscout-site-6ah.11`

- Copy `podman-autostart.ps1` into `deploy/windows/` and parameterize it (service list, timeouts).
- Write `register-autostart.ps1`, which creates or updates the `PodmanMachineAutostart` scheduled task idempotently.
- Write `configuration.dsc.yaml` for `winget configure`: Podman, Docker Compose, Git, Go, Node LTS, and age/sops.
- Update PODMAN_MIGRATION.md to point at these files instead of describing hand-made state.

**Done when:** a fresh Windows machine reaches a working dev stack with two commands.

### Phase 11: Docs and cleanup (S) · `groupscout-site-6ah.12`

- Write `backend/docs/guides/INFRA.md`, an operator runbook covering day-2 tasks: deploy, rollback, rotate a secret, add a hostname, restore.
- Update HOME_DEPLOY.md and COOLIFY.md headers to "superseded by IaC path" or "alternative," keeping their content for reference.
- Update `WHERE_TO_START.md` and `ROADMAP.md`.
- Add `bd remember` notes for any non-obvious gotchas found during implementation.

---

## 8. Sequencing

```
Phase 0 ──► Phase 1 ──► Phase 2 ──┐
               │                   ├──► Phase 5 ──► Phase 7 ──► Phase 8 ──► Phase 11
               ├──► Phase 3 ───────┤        ▲
               └──► Phase 4 ───────┘        │
                                   Phase 6 ─┘ (needs D1/D2/D5/D9)
Phase 9 can start after Phase 2 (heartbeat) and finish after Phase 6 (monitors).
Phase 10 is independent and can start any time.
```

Suggested first slice (about a week): **Phase 1 + the `n8n-init` part of Phase 2 + Phase 3 + the cadence heartbeat from Phase 9.** These fix the incident classes above without committing to a deployment target.

---

## 9. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Losing the age key makes every secret unrecoverable | Password-manager entry plus an offline copy, with a recovery test in Phase 3 |
| OpenTofu state leaks tunnel tokens | State encryption plus a private bucket with a scoped key |
| `n8n-init` re-import wipes the cadence `staticData` and causes duplicate sends | Import only on a checksum change, and test explicitly in Phase 2 |
| The n8n CLI changes between versions | Pin the version, and have Renovate PRs run the empty-volume import test |
| Docker port publishing bypasses UFW | Prod Compose publishes nothing on `0.0.0.0`, and Phase 5 verifies with an external port scan |
| Over-engineering a single-host internal tool | Phases 1–4 and 9 are valuable on their own, and Phases 5–7 wait for a real D1 decision |
| Two copies of the n8n workflow JSON drift apart | One canonical location (D8 / Phase 2), with a CI check that compares checksums if both remain |
| Promtail-to-Alloy migration breaks the log dashboards | Run them side by side for one week before removing Promtail |

---

## 10. Open Questions

- Is the backend GitHub repo public or private? This affects GHCR visibility, self-hosted runner safety, and Actions minutes.
- Which domain should be used? (The `EMAIL_FROM` example mentions `groupscout.ai`, and Resend domain verification is `7ak`.) Cloudflare Tunnel needs a zone on Cloudflare.
- Who besides you needs access to n8n, Grafana, and the UI? This sets the Cloudflare Access allow-list.
- ~~Is `SENDGRID_API_KEY` still used?~~ No. No Go code reads it, and Phase 1 removed it from Compose.
- Does `alertd` expose `/metrics`? If not, should it?
- Is a home GPU box for Ollama (D10) worth keeping powered on around the clock?

---

## Appendix A: Secrets Inventory

| Secret | Consumers | Notes |
|---|---|---|
| `CLAUDE_API_KEY` | groupscout, alertd | Rotate if it was ever in a plaintext `.env` backup |
| `SLACK_WEBHOOK_URL` | groupscout, n8n (`GROUPSCOUT_OPS_SLACK_WEBHOOK_URL`) | |
| `SLACK_BOT_TOKEN` | alertd | |
| `RESEND_API_KEY` | groupscout | |
| ~~`SENDGRID_API_KEY`~~ | none | Unused. Removed from Compose in Phase 1; delete it from `.env` |
| `API_TOKEN` | groupscout, n8n (`GROUPSCOUT_API_TOKEN`), UI proxy? | Rotate when moving to production |
| `HUNTER_API_KEY` | planned | |
| `POSTGRES_PASSWORD` | postgres, groupscout, alertd | **New.** Currently hard-coded as `groupscout` |
| `N8N_ENCRYPTION_KEY` | n8n | **New.** Currently auto-generated inside the volume |
| `GF_SECURITY_ADMIN_PASSWORD` | grafana | **New.** Currently the default |
| `CLOUDFLARE_TUNNEL_TOKEN` | cloudflared | **New.** OpenTofu output |
| `CLOUDFLARE_API_TOKEN` | OpenTofu only | Scoped to zone, tunnel, and Access |
| `HCLOUD_TOKEN` | OpenTofu only | Only if D1 = c |
| `TS_AUTHKEY` | Ansible (host), CI (ephemeral) | Tailscale |
| `RESTIC_PASSWORD`, bucket keys | backups | **New** |
| `GHCR_READ_TOKEN` | host image pulls | Only if the packages are private |
| Tofu state passphrase | OpenTofu | **New** |
| age private key | humans and CI | Root of trust. Never in git |

Not secrets (plain config): `RICHMOND_PERMITS_URL`, `DELTA_PERMITS_URL`, collector toggles, thresholds, `OLLAMA_*`, `AUDIT_RETENTION_*`, `LEAD_NOTIFY_EMAILS`, `EMAIL_FROM`.

## Appendix B: Port Exposure

| Service | Today (host) | Proposed production |
|---|---|---|
| groupscout API | `0.0.0.0:8080` | Tunnel hostname `api.<domain>`, protected by bearer token |
| UI | `3001`/`3002` | Tunnel `app.<domain>` + Cloudflare Access |
| alertd | `0.0.0.0:8081` | Internal only, or Tunnel + Access |
| n8n | `0.0.0.0:5678` | Tunnel `n8n.<domain>` + Access (only if webhooks need it), otherwise Tailscale only |
| Grafana | `0.0.0.0:3000` | Tunnel `grafana.<domain>` + Access |
| Postgres | `0.0.0.0:5432` | **None.** Admin access through an SSH tunnel over Tailscale |
| Prometheus | `0.0.0.0:9090` | None (internal network) |
| Loki | `0.0.0.0:3100` | None (internal network) |
| Ollama | Not published ✅ | None |
