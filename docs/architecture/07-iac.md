# 7. Infrastructure as Code

_As of 2026-10-16. The full plan is [INFRASTRUCTURE_AS_CODE.md](../../backend/docs/planning/INFRASTRUCTURE_AS_CODE.md); the epic is `groupscout-site-6ah`._

## 7.1 Layers and phase status

```mermaid
flowchart TB
    subgraph L5["L5 Monitoring as code"]
        p9["Phase 9: Healthchecks heartbeat, ops metrics,<br/>Grafana alert rules<br/>BUILT + LIVE, PR #15 open; owner setup open (6ah.16)"]
    end
    subgraph L4["L4 Delivery"]
        p4["Phase 4: GitHub Actions CI, GHCR images, Renovate<br/>MERGED; branch protection + Renovate app open (6ah.14)"]
        p7["Phase 7: deploy pipeline with verify + auto-rollback<br/>PLANNED (6ah.8)"]
    end
    subgraph L3["L3 App config"]
        p2["Phase 2: n8n workflows, Prometheus, Grafana, Alloy config in git<br/>MERGED"]
    end
    subgraph L2["L2 Workload"]
        p1["Phase 1: pinned images, healthchecks, profiles,<br/>prod overlay, non-root image<br/>MERGED"]
    end
    subgraph L1["L1 Host config"]
        p5["Phase 5: Ansible for the groupscout-prod WSL host, rootless Podman<br/>BUILT, PR #17 open; staging running; cutover open (6ah.18)"]
        p10["Phase 10: Windows workstation as code (winget DSC, autostart task)<br/>BUILT, PR #16 open; autostart task live"]
    end
    subgraph L0["L0 Resources"]
        p6["Phase 6: OpenTofu for ingress, DNS, backup bucket, monitors<br/>PLANNED (6ah.7); needs D5 ingress (6ah.17)"]
    end
    subgraph X["Cross-cutting"]
        p3["Phase 3: secrets as code, SOPS + age, gitleaks<br/>MERGED; age key backup open (6ah.13)"]
        p8["Phase 8: backups and disaster recovery<br/>PLANNED (6ah.9)"]
        p11["Phase 11: runbook and docs cleanup<br/>PLANNED (6ah.12)"]
    end
    p0["Phase 0: decisions D1-D11<br/>DONE"] --> L2
    L0 --> L1 --> L2 --> L3 --> L4 --> L5
    X -.-> L1
    X -.-> L4

    classDef done fill:#d8f3dc,stroke:#2d6a4f,color:#081c15
    classDef built fill:#fff3bf,stroke:#b08900,color:#3d2c00
    classDef planned fill:#e9ecef,stroke:#6c757d,color:#212529
    class p0,p1,p2,p3,p4 done
    class p5,p9,p10 built
    class p6,p7,p8,p11 planned
```

Green: merged to `main`. Yellow: built and in use, waiting on a PR or an owner step. Grey: planned.

Decisions that shape the diagram: D1 production on a dedicated WSL 2 distro; D2 OpenTofu; D3 Ansible; D4 rootless Podman + Compose; D5 ingress **open**; D6 no Coolify; D7 SOPS + age; D8 IaC lives in the backend repo's `deploy/`; D10 Ollama off in production; D11 keep n8n.

## 7.2 Secrets flow

```mermaid
flowchart LR
    key["age private key<br/>~/.config/sops/age/keys.txt (0600)<br/>backup: password manager + offline (6ah.13)"]
    enc["deploy/secrets/prod.sops.env<br/>values encrypted, committed to git"]
    edit["make secrets-edit"]
    render["make secrets-render"]
    env[".env (0600, gitignored)<br/>groupscout-iac worktree today"]
    compose["podman compose<br/>interpolates ${VAR}"]
    containers["containers: only the variables<br/>each service lists"]
    ansible["Ansible community.sops lookup<br/>(WSL host)"]
    hostenv["/opt/groupscout/.env (0600)<br/>on groupscout-prod"]
    scan["gitleaks: make secrets-scan,<br/>secret-scan.yml in CI,<br/>GitHub push protection"]

    edit --> enc
    key --> render
    enc --> render --> env --> compose --> containers
    key --> ansible
    enc --> ansible --> hostenv --> compose
    scan -.->|"blocks plaintext secrets"| enc
```

## 7.3 CI and image publishing

```mermaid
flowchart LR
    pr["Pull request or push to main"] --> go["go job<br/>gofmt, vet, unit tests, eval gate"]
    pr --> integ["integration job<br/>pgvector service, -p 1"]
    pr --> comp["compose job<br/>make compose-check"]
    pr --> ans["ansible job<br/>ansible-lint, syntax check (PR #17)"]
    pr --> ss["secret-scan.yml<br/>gitleaks, full history"]
    go --> img
    integ --> img
    comp --> img
    img["image job<br/>build, start with prod overlay,<br/>/health must report this SHA,<br/>grype: fail on fixable critical"] -->|"main only"| ghcr["ghcr.io/alvindcastro/groupscout<br/>:full-sha and :main"]
    reno["Renovate (weekly)<br/>Go modules, action SHAs, image digests"] -.->|"PRs"| pr
```

The UI repo runs the same shape: `test` (`node:test`), then `image` to `ghcr.io/alvindcastro/groupscout-ui`, plus its own gitleaks scan.

## 7.4 Delivery: today and planned

```mermaid
flowchart TB
    subgraph today["Today (manual, Podman machine)"]
        t1["merge main into task/iac-phase9-monitoring<br/>in the groupscout-iac worktree"] --> t2["podman compose build groupscout<br/>GROUPSCOUT_VERSION=short sha"]
        t2 --> t3["tag the old image groupscout:old-sha<br/>(rollback)"]
        t3 --> t4["podman compose up -d --no-build --no-deps groupscout"]
        t4 --> t5["check /health version and logs"]
    end
    subgraph cutover["After the WSL cutover (6ah.18)"]
        c1["set groupscout_image_tag to a main SHA<br/>in deploy/ansible group_vars"] --> c2["make ansible-apply<br/>pulls the GHCR image, renders .env, compose up"]
        c2 --> c3["make ansible-check for drift"]
    end
    subgraph planned["Phase 7 (planned)"]
        d1["deploy workflow on main"] --> d2["SSH over Tailscale (D5)"] --> d3["pull SHA, compose up"] --> d4{"verify: /health SHA,<br/>smoke checks"}
        d4 -->|fail| d5["auto-rollback to previous SHA"]
    end
    today -.-> cutover -.-> planned
```

The live image comes from `task/iac-phase9-monitoring`, not `main`, until PR #15 merges. Check the running container's `org.opencontainers.image.revision` label before deploying.
