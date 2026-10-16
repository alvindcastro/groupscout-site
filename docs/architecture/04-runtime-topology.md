# 4. Runtime Topology

_As of 2026-10-16._

## 4.1 Today: the Podman machine on the home PC

The live stack runs in the Podman machine on the Windows PC. Compose project `groupscout`, working directory `C:\Users\alvin\GolandProjects\groupscout-iac` (branch `task/iac-phase9-monitoring`), files `docker-compose.yml` + `docker-compose.override.yml`, and `.env` rendered from `deploy/secrets/prod.sops.env`.

```mermaid
flowchart TB
    subgraph pc["Windows 11 PC"]
        task["Scheduled task PodmanMachineAutostart (at logon)<br/>%LOCALAPPDATA%\GroupScout\podman-autostart.ps1"]
        browser["Browser: localhost:3000 Grafana,<br/>:5678 n8n, :9090 Prometheus"]

        subgraph devdistro["WSL distro: Ubuntu (dev)"]
            dev["Claude Code, Go 1.26, Node 22,<br/>sops + age key, gitleaks,<br/>Ansible controller"]
        end

        subgraph pm["Podman machine: podman-machine-default (WSL 2)"]
            subgraph net["bridge network groupscout_net"]
                app["groupscout_app<br/>Go API + pipeline<br/>:8080"]
                pg[("groupscout_postgres<br/>pgvector 0.8.7 / PG 17<br/>:5432")]
                n8n["groupscout_n8n<br/>n8n 2.43.5<br/>:5678"]
                prom["groupscout_prometheus<br/>v3.15.0 :9090 (obs)"]
                graf["groupscout_grafana<br/>13.2.4 :3000 (obs)"]
                loki["groupscout_loki<br/>3.7.8 :3100 (obs)"]
                alloy["groupscout_alloy<br/>v1.20.1 (obs)"]
                ollama["groupscout_ollama<br/>(llm profile; app has OLLAMA_ENABLED=false)"]
                alertd["groupscout_alertd :8081<br/>defined, not running"]
            end
            vols[("volumes: pgdata, n8n_data,<br/>prometheus_data, grafana_data,<br/>alloy_data, groupscout_ollama_data")]
            sock["Podman API socket"]
        end
    end

    task -->|"podman machine start, then compose up<br/>postgres first, then app, n8n, obs"| pm
    n8n -->|"POST /run"| app
    app --> pg
    prom -->|"scrape /metrics"| app
    alloy -->|"container discovery"| sock
    alloy -->|"push logs"| loki
    graf --> prom
    graf --> loki
    pg --- vols
    dev -->|"podman.exe via WSL interop<br/>(or SSH-forwarded socket)"| pm
    browser --> graf
    browser --> n8n
```

- Host ports come from `docker-compose.override.yml` and are published on the PC's localhost only. Nothing is reachable from outside the PC (ingress, decision D5, is open).
- The Podman machine does not start on boot by itself; the logon task starts it, waits for Postgres to be healthy, then starts the rest.
- Deploying today means building `groupscout:local` on the machine from the `groupscout-iac` worktree and recreating `groupscout_app` (see [07-iac.md](./07-iac.md#74-delivery-today-and-planned)).

## 4.2 Target: the `groupscout-prod` WSL host (Phase 5)

Built on PR #17 and running as staging. The cutover is `groupscout-site-6ah.18`.

```mermaid
flowchart TB
    subgraph pc["Windows 11 PC (all WSL 2 distros share one network namespace)"]
        logon["Windows logon task<br/>starts groupscout-prod and holds a keep-alive session"]

        subgraph devdistro["WSL distro: Ubuntu (dev / Ansible controller)"]
            ansible["ansible-core 2.21.4<br/>make ansible-apply / ansible-check"]
            sops["community.sops lookup<br/>decrypts prod.sops.env"]
        end

        subgraph prod["WSL distro: groupscout-prod (Ubuntu 26.04, systemd)"]
            sshd["sshd 127.0.0.1:2222<br/>key only"]
            subgraph deployuser["user deploy (uid 2000), lingering"]
                unit["systemd user unit groupscout.service<br/>podman compose up at boot"]
                stack["rootless Podman 5.7 + Compose<br/>/opt/groupscout: compose files, config, .env (0600)<br/>app, postgres, n8n, obs: same images as today"]
            end
        end

        pm["Podman machine<br/>(development only after cutover)"]
    end

    ansible -->|"SSH 127.0.0.1:2222"| sshd
    sops --> ansible
    sshd --> deployuser
    unit --> stack
    logon --> prod
```

Roles in `deploy/ansible/site.yml`: `base`, `hardening`, `podman`, `groupscout`. The host disables WSL interop and detaches `binfmt_misc` so stopping it cannot break the other distros, and uses uid 2000 because the Podman machine already owns uid 1000's cgroup.
