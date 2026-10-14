# Infrastructure as Code: Brainstorm & Task Plan

Date: 2026-10-14
Status: Draft brainstorm. No decisions are final until Phase 0 is ticked.

> **Tracking note:** Beads is the source of truth for active work. The checkboxes below help plan the brainstorm. Once a phase is approved, file it as a Beads epic with child issues and track progress there, not here. The existing home deploy, backups, and restore-smoke work is `groupscout-site-39g`. Phases 5–8 below would replace its manual runbook steps with code.

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

- [ ] Rebuild the production host from zero in **under 1 hour** with one command plus the secrets key.
- [ ] Trace every running container to a git commit (image tag = SHA, `/health` shows it).
- [ ] Make it impossible for the lead cadence to stop silently (workflow-as-code plus an external heartbeat).
- [ ] Keep secrets encrypted at rest in git, so nothing lives only in a local `.env`.
- [ ] Make drift detectable, so a scheduled plan/check reports differences.
- [ ] Use one IaC codebase for both candidate targets: a home Linux box and a Hetzner VPS.

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

Each decision lists the options, a recommendation, and the reason. Tick the box once a decision is final, and record any change from the recommendation.

### D1: Deployment target

- (a) Stay on the Windows workstation with the Podman machine. Uptime depends on a desktop and on user logon.
- (b) A home Linux box with Cloudflare Tunnel. $0, and the current `39g` path.
- (c) A Hetzner VPS. About $10/month and the best uptime.

**Recommendation:** Write L1–L5 against an abstract target, "one Debian/Ubuntu host reachable over SSH." Targets (b) and (c) then share 100% of the code, and only the Ansible inventory and the L0 module differ. Pick the first real target separately. Keep (a) for development only.

- [ ] D1 decided: ________

### D2: Provisioning tool (L0)

- OpenTofu, Terraform, Pulumi (Go), or none.

**Recommendation:** **OpenTofu.** It's MPL-licensed, uses the same `hcloud` and `cloudflare` providers, and has **native client-side state encryption**. That matters here because state will hold tunnel tokens. Pulumi in Go is a reasonable alternative if you prefer writing Go over HCL. "None" is valid for a home-only target, where L0 shrinks to Cloudflare resources plus a backup bucket.

- [ ] D2 decided: ________

### D3: Host configuration (L1)

- Ansible, cloud-init only, or NixOS.

**Recommendation:** **Ansible.** Runs are idempotent, `--check --diff` gives you drift detection, and it works on a home box that has no cloud-init. On Hetzner, cloud-init only creates the deploy user and SSH key, and Ansible does the rest. NixOS is powerful, but it's a big learning step for one host.

- [ ] D3 decided: ________

### D4: Container runtime on the server

- Docker Engine + Compose, or Podman + Quadlets (systemd units).

**Recommendation:** **Docker Engine + Compose for v1.** It's the documented production baseline, and existing Traefik, Promtail, and smoke scripts assume it. On Linux, the Docker daemon starts at boot, so `restart: unless-stopped` works there, unlike on the Podman machine. Revisit Quadlets later; `podlet` can convert the Compose file.
**Gotcha:** Docker-published ports **bypass UFW**. In production, bind ports to `127.0.0.1` or don't publish them at all.

- [ ] D4 decided: ________

### D5: Ingress

- Cloudflare Tunnel, Traefik + Let's Encrypt + port forwarding, or Tailscale only.

**Recommendation:** **Cloudflare Tunnel** with the tunnel, DNS, and Access policies defined in OpenTofu. This means no open inbound ports on either target and no CGNAT problems. Put **Cloudflare Access** (free for small teams) in front of n8n, Grafana, the UI, and alertd. Expose only the API hostname publicly, protected by its bearer token. Use **Tailscale** for SSH and admin access.

- [ ] D5 decided: ________

### D6: Coolify

**Recommendation:** **Drop it on the IaC path.** Coolify keeps its config in its own database and UI, which conflicts with git being the source of truth, and it has a recent CVE history. Keep COOLIFY.md as the click-ops alternative.

- [ ] D6 decided: ________

### D7: Secrets

- SOPS + age, Ansible Vault, 1Password or Bitwarden Secrets Manager, or Infisical/Doppler.

**Recommendation:** **SOPS + age.** It's free and git-native, and it integrates with Ansible (`community.sops`) and OpenTofu (sops provider). It encrypts values, not whole files, so diffs stay readable. The age private key is the root of trust: store it in a password manager and keep one offline backup.

- [ ] D7 decided: ________

### D8: Where the IaC code lives

- (a) A `deploy/` folder in the backend repo, or (b) a new `groupscout-infra` repo.

**Recommendation:** **(a) to start.** Compose already lives in the backend repo, so one PR can change the app and its infrastructure together, and one CI setup covers both. The UI repo publishes an image to GHCR that the backend's `deploy/` consumes by tag. Long-lived docs stay in this coordination repo, following the existing convention. Split into a separate repo only if a second app or a different deploy cadence appears.

- [ ] D8 decided: ________

### D9: OpenTofu state backend

**Recommendation:** An S3-compatible bucket (Cloudflare R2, Hetzner Object Storage, or Backblaze B2) with OpenTofu state encryption turned on. The state bucket is created once by hand, or by a tiny `bootstrap/` stack with local state, to solve the chicken-and-egg problem. Document that step.

- [ ] D9 decided: ________

### D10: Ollama in production

- Disable it on the VPS (Claude-only enrichment), buy a bigger or GPU box, or keep Ollama on a home GPU machine and reach it over Tailscale.

**Recommendation:** Move Ollama into an `llm` Compose profile that's **off by default in production**, and set `OLLAMA_ENABLED=false` there. Revisit after measuring how often the Ollama paths actually matter. Hetzner GPU servers cost far more than the rest of the stack combined.

- [ ] D10 decided: ________

### D11: Scheduling

- Keep n8n with workflow-as-code, or move the cadence to a systemd timer or the in-app scheduler.

**Recommendation:** Keep n8n for now and add `n8n-init` (Phase 2). Moving the cadence would be a product change, not an infrastructure change. Track it separately if it's wanted.

- [ ] D11 decided: ________

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
├── docker-compose.yml              # dev entrypoint kept for Makefile/scripts; `include:`s deploy/compose
├── deploy/
│   ├── compose/
│   │   ├── compose.base.yml        # services, networks, volumes, healthchecks (no host ports)
│   │   ├── compose.dev.yml         # host port publishing, local builds
│   │   ├── compose.prod.yml        # GHCR images by tag, 127.0.0.1 binds, limits, logging
│   │   ├── compose.obs.yml         # profile: prometheus, grafana, loki, alloy
│   │   └── compose.llm.yml         # profile: ollama + ollama-init
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

Sizes: **S** is about one session, **M** is a few sessions, and **L** is a week or more of evenings. Phases 1–4 don't depend on D1 and can start right away.

### Phase 0: Decisions and groundwork (S)

- [ ] Review D1–D11 and record the outcome of each in §5.
- [ ] Mark PHASES.md / PROMPTS_PHASE25 Part D (GCP Terraform) as **superseded by this doc**.
- [ ] Add a pointer from DEPLOYMENT_OPTIONS.md and the ROADMAP Phase 14/26 entries to this doc.
- [ ] Create a Beads epic "Infrastructure as Code" with one child issue per approved phase, and link `39g` as dependent on Phases 5–8.
- [ ] Create a backend task branch (e.g. `task/iac-foundation`).

**Done when:** the decisions are recorded and the Beads epic exists.

### Phase 1: Harden the existing Compose and images (M), no new tools

Compose:
- [ ] Add a top-level `name: groupscout` so `-p groupscout` is no longer needed with Docker or Podman.
- [ ] Pin every image to an explicit version (record what's running now, e.g. n8n `2.14.2`, then pin), and later add digests through Renovate.
- [ ] Pin `golang:1.26-alpine` and `alpine:latest` in the backend Dockerfile to concrete releases.
- [ ] Replace the hard-coded Postgres password with `${POSTGRES_PASSWORD:?required}`, and build `DATABASE_URL` from variables.
- [ ] Set `N8N_ENCRYPTION_KEY`, `GENERIC_TIMEZONE=America/Vancouver`, and a Grafana admin password from env.
- [ ] Replace `env_file: .env` on `alertd` with an explicit list of the variables it needs.
- [ ] Add healthchecks for `groupscout` (`/health`), `alertd`, and `n8n` (`/healthz`).
- [ ] Add a default `logging:` block (json-file, `max-size: 10m`, `max-file: 5`) so logs can't fill the disk.
- [ ] Split services into profiles: `core` (postgres, groupscout, alertd, n8n), `obs`, `llm`, `ui`.
- [ ] Split into base/dev/prod files (§6). Dev keeps today's host ports; prod publishes nothing except `127.0.0.1` binds where needed.
- [ ] Verify `docker compose config` and `podman compose config` both render for each profile combination.

Images:
- [ ] Inject the version at build time (`-ldflags "-X main.version=$SHA"`) and return it from `/health`.
- [ ] Add OCI labels (`org.opencontainers.image.revision`, `.source`, `.created`).
- [ ] Run as a non-root user in the backend runtime image.
- [ ] UI: make the `production` stage start from a clean base instead of `FROM test`, so test files don't ship.

**Done when:** `make docker-up` still works locally, the smoke scripts pass, `/health` shows the SHA, and no `:latest` remains.

### Phase 2: App config as code (M)

n8n:
- [ ] Copy the workflow JSON to `deploy/config/n8n/workflows/`. Decide whether the coordination repo copy becomes the doc reference or a sync target, and remove the duplication risk.
- [ ] Add an `n8n-init` service (same pinned image and volume) that runs **before** `n8n` (`depends_on: service_completed_successfully`):
  - [ ] Import each workflow **only if it's missing or its checksum changed**, to preserve the `staticData` cadence-dedupe keys.
  - [ ] Activate or publish the workflow using the command that matches the pinned n8n version.
  - [ ] Exit non-zero if import or activation fails, so `n8n` doesn't start in a broken state.
- [ ] Remove the manual "import, activate, restart" steps from PODMAN_MIGRATION.md and N8N_GUIDE.md once `n8n-init` is verified.
- [ ] Test: start with an empty `n8n_data` volume and confirm the cadence workflow is active with no manual steps. This reproduces the Podman incident.

Observability:
- [ ] Add `deploy/config/prometheus/prometheus.yml` that scrapes `groupscout:8080/metrics`, plus `alertd` if it exposes metrics.
- [ ] Add Grafana provisioning: Prometheus and Loki datasources, plus a starter "GroupScout overview" dashboard JSON (run counts, lead counts, collector errors, HTTP latency).
- [ ] Replace Promtail with **Grafana Alloy**, using a config that works for Docker (socket) and is documented for Podman (journald or file).
- [ ] Mount all config read-only, so nothing important lives only in the `grafana_data` or `prometheus_data` volumes.

**Done when:** `down -v` followed by `up` brings back the active workflow, the scrape targets, and the dashboards with zero manual steps.

### Phase 3: Secrets as code (S)

- [ ] Generate an age keypair. Store the private key in a password manager plus one offline backup, and document the recovery steps.
- [ ] Add `.sops.yaml` with creation rules for `deploy/secrets/*.sops*` and `ansible/**/secrets.sops.yml`.
- [ ] Build `deploy/secrets/prod.env.sops` from the inventory in Appendix A. Don't commit the plaintext `.env`.
- [ ] Add `make secrets-edit` (`sops deploy/secrets/prod.env.sops`) and `make secrets-render`, which decrypts to a `0600` tmpfs or ignored path.
- [ ] Add a pre-commit hook or CI check that fails if anything matching `*.env` (not `.sops`), `*.pem`, or a token pattern gets committed (e.g. gitleaks).
- [ ] Rotate secrets that were ever shared in plaintext or are defaults: the Postgres password, the Grafana admin password, and `API_TOKEN`.

**Done when:** a fresh clone plus the age key can render a working `.env`, and the secret scanner runs in CI.

### Phase 4: CI with build, scan, and publish (M)

- [ ] `ci.yml` (backend): `go vet`, `go test ./...`, `make eval-gate`, `docker build`, and `docker compose config` for each profile, on every PR.
- [ ] `ci.yml` (UI): `npm test` plus the production-target image build on every PR.
- [ ] `images.yml`: on merge to `main`, build and push `ghcr.io/<owner>/groupscout:<sha>` and `groupscout-ui:<sha>`, plus a moving `main` tag.
- [ ] Image vulnerability scan (Trivy or Grype) that fails on fixable critical findings.
- [ ] Pin every third-party Action to a commit SHA.
- [ ] Add `renovate.json` covering Docker digests in Compose and Dockerfiles, GitHub Actions SHAs, OpenTofu providers, Ansible collections, and Go modules (grouped and scheduled weekly).
- [ ] Branch protection on `main`: CI must pass before merging.
- [ ] Add a backend-owned integration-test job with a Postgres service container, keeping the advisory-lock serialization that already exists.

**Done when:** a broken Dockerfile can't merge, and every `main` commit has a pullable image tagged with its SHA.

### Phase 5: Host configuration with Ansible (L), needs D1/D3/D4

- [ ] Write the inventory for the chosen target(s) and `group_vars` (domain, timezone, deploy user, backup bucket).
- [ ] Role `base`: packages, timezone, `unattended-upgrades`, chrony/NTP, swap file sized for 8 GB hosts.
- [ ] Role `hardening`: deploy user with an SSH key only, no root login, no password auth, UFW default-deny inbound, SSH allowed only on the Tailscale interface.
- [ ] Role `tailscale`: install and join with an auth key from SOPS, and tag the node `tag:groupscout`.
- [ ] Role `docker`: install Docker Engine from the official repo, `daemon.json` (log rotation, `live-restore`), and add the deploy user to the `docker` group.
- [ ] Role `groupscout`: create `/opt/groupscout`, render the Compose files and decrypted `.env` (`0600`), log in to GHCR with a read-only token, then `docker compose up -d` with the prod and profile flags.
- [ ] Role `backups`: install restic and the systemd units from Phase 8.
- [ ] Make `site.yml` idempotent: a second run reports `changed=0`.
- [ ] Use `ansible-lint` in CI, and run `ansible-playbook --check --diff` as the drift check.

**Done when:** one command takes a fresh Debian/Ubuntu host to a running, healthy stack, and a re-run makes no changes.

### Phase 6: Cloud resources with OpenTofu (M), needs D1/D2/D5/D9

- [ ] `bootstrap/`: state bucket and a scoped API key, using local state (commit only the encrypted state or none at all).
- [ ] Configure `envs/prod` backend and state encryption (passphrase or key from SOPS).
- [ ] Module `cloudflare_edge`: tunnel, tunnel ingress config (api, ui, n8n, grafana hostnames), DNS records, and Access application + policy (allow-list emails) for every hostname except the API. Output the tunnel token into SOPS or an Ansible var.
- [ ] Module `hetzner_host` (only if D1 = c): server (CX32, Ubuntu LTS), firewall that blocks all inbound since the tunnel and Tailscale are outbound, SSH key, cloud-init that creates the deploy user, and a daily-backup flag.
- [ ] Module `backup_bucket`: bucket with a lifecycle rule and a write-only key for restic.
- [ ] Module `monitors` (L5): uptime check on the public `/health`, plus the cadence heartbeat check (Phase 9).
- [ ] Use `tofu fmt -check`, `tofu validate`, and `tflint` in CI, and post `tofu plan` as a PR comment.

**Done when:** `tofu apply` creates the edge and host resources from scratch, and `tofu plan` on an unchanged tree shows no changes.

### Phase 7: Deploy pipeline (M)

- [ ] `deploy/scripts/deploy.sh <sha>`: take a pre-deploy `pg_dump`, set the image tag, `docker compose pull`, `docker compose up -d --remove-orphans`, run `verify.sh`, and roll back automatically to the previous tag if the check fails.
- [ ] `verify.sh`: `/health` returns 200 with `database: ok` and the expected SHA, `alertd` is healthy, the n8n workflow is active, and the UI `/healthz` returns 200.
- [ ] `deploy.yml`: manually triggered (or triggered on a release tag), with GitHub Environment `production` requiring approval. The runner joins Tailscale with an ephemeral key, then SSHes in and runs `deploy.sh <sha>`.
- [ ] Record each deploy (SHA, time, actor) in a `deploys.log` on the host or as a GitHub deployment.
- [ ] Document rollback: `deploy.sh <previous-sha>`.
- [ ] Confirm the app runs DB migrations on startup safely, and document the expand/contract rule for destructive migrations.

**Done when:** a merge to `main` followed by one approved click puts a verified build in production, and a bad build rolls itself back.

### Phase 8: Backups and disaster recovery (M), closes the backup part of `39g`

- [ ] Write `backup.sh`: `pg_dump -Fc` plus the n8n volume (and Grafana, if anything there isn't provisioned) into restic, sent to the offsite bucket.
- [ ] Add a systemd timer that runs nightly, with retention of 7 daily, 4 weekly, and 6 monthly.
- [ ] Have the backup ping a heartbeat check when it succeeds, so a failed or missed backup raises an alert.
- [ ] Write `restore-smoke.sh`: restore the latest snapshot into a throwaway Postgres container, then run sanity queries (row counts on `leads` and `raw_inputs`, and confirm the `vector` extension is present).
- [ ] Run the restore smoke monthly with a timer and once by hand now, recording the result in `39g`.
- [ ] Write a DR runbook (in this repo) covering a total host loss: new host, `tofu apply`, `ansible-playbook site.yml`, restore, verify. Time the drill against the 1-hour goal.
- [ ] Retire or rewrite `scripts/backup-volumes.sh` (which is Docker-only and local-only) in favour of the above.

**Done when:** a timed DR drill rebuilds production from nothing in under an hour.

### Phase 9: Monitoring as code (S)

- [ ] **Cadence dead-man's switch:** the n8n workflow (or the backend `/run` path) pings a heartbeat URL after each successful cadence run. The check expects a ping on Sun, Tue, and Thu by about 18:30 Pacific and alerts if one doesn't arrive. This is the check that would have caught the Podman/n8n silent failure.
- [ ] External uptime check on the public `/health`, alerting through email or Slack.
- [ ] Grafana alert rules, provisioned as code: collector error-rate spike, no leads in N days, Postgres disk above 80%, a container restart loop.
- [ ] Route alerts to a Slack channel separate from the lead notifications.

**Done when:** stopping the n8n container through a cadence window produces an alert without anyone checking manually.

### Phase 10: Dev workstation as code (S), Windows/Podman

- [ ] Copy `podman-autostart.ps1` into `deploy/windows/` and parameterize it (service list, timeouts).
- [ ] Write `register-autostart.ps1`, which creates or updates the `PodmanMachineAutostart` scheduled task idempotently.
- [ ] Write `configuration.dsc.yaml` for `winget configure`: Podman, Docker Compose, Git, Go, Node LTS, and age/sops.
- [ ] Update PODMAN_MIGRATION.md to point at these files instead of describing hand-made state.

**Done when:** a fresh Windows machine reaches a working dev stack with two commands.

### Phase 11: Docs and cleanup (S)

- [ ] Write `backend/docs/guides/INFRA.md`, an operator runbook covering day-2 tasks: deploy, rollback, rotate a secret, add a hostname, restore.
- [ ] Update HOME_DEPLOY.md and COOLIFY.md headers to "superseded by IaC path" or "alternative," keeping their content for reference.
- [ ] Update `WHERE_TO_START.md` and `ROADMAP.md`.
- [ ] Add `bd remember` notes for any non-obvious gotchas found during implementation.

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

- [ ] Is the backend GitHub repo public or private? This affects GHCR visibility, self-hosted runner safety, and Actions minutes.
- [ ] Which domain should be used? (The `EMAIL_FROM` example mentions `groupscout.ai`, and Resend domain verification is `7ak`.) Cloudflare Tunnel needs a zone on Cloudflare.
- [ ] Who besides you needs access to n8n, Grafana, and the UI? This sets the Cloudflare Access allow-list.
- [ ] Is `SENDGRID_API_KEY` still used, or can it be dropped now that Resend handles email?
- [ ] Does `alertd` expose `/metrics`? If not, should it?
- [ ] Is a home GPU box for Ollama (D10) worth keeping powered on around the clock?

---

## Appendix A: Secrets Inventory

| Secret | Consumers | Notes |
|---|---|---|
| `CLAUDE_API_KEY` | groupscout, alertd | Rotate if it was ever in a plaintext `.env` backup |
| `SLACK_WEBHOOK_URL` | groupscout, n8n (`GROUPSCOUT_OPS_SLACK_WEBHOOK_URL`) | |
| `SLACK_BOT_TOKEN` | alertd | |
| `RESEND_API_KEY` | groupscout | |
| `SENDGRID_API_KEY` | groupscout | Possibly unused (see open questions) |
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
