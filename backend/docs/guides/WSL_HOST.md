# WSL Production Host

IaC Phase 5 (`groupscout-site-6ah.6`). Production is moving off the Podman machine to a dedicated WSL 2 distro, **`groupscout-prod`** (Ubuntu 26.04), configured by Ansible like any Ubuntu host, with rootless Podman 5.7 + Compose.

- Decision D1 (2026-10-15): run production in WSL. It's $0, and the same Ansible moves to a VPS or home Linux box later by changing the inventory. A VPS is still the option for running while the PC is off.
- Decision D4 (2026-10-15): use Podman over Docker where possible.

**Status (2026-10-15):** built on backend PR [#17](https://github.com/alvindcastro/groupscout/pull/17), stacked on #16.
- `groupscout-prod` exists and runs the stack as **staging**: Postgres and the API on port 18080, throwaway secrets, no n8n.
- The live stack is still on the Podman machine. The cutover is `groupscout-site-6ah.18`.

## Layout

| Piece | Where |
|---|---|
| Create the distro, ready for Ansible | backend `deploy/windows/new-wsl-host.ps1` (Windows PowerShell) |
| Configure the host and run the stack | backend `deploy/ansible/site.yml`: roles `base`, `hardening`, `podman`, `groupscout` |
| Inventory | `deploy/ansible/inventory/home-wsl.yml`: `groupscout-prod` at `127.0.0.1:2222`, user `deploy`, key `~/.ssh/groupscout_prod_ed25519` |
| Per-host overrides | `deploy/ansible/inventory/host_vars/groupscout-prod.yml` (the staging settings until cutover) |
| Secrets | `deploy/secrets/prod.sops.env` (production) and `staging.sops.env` (throwaway values, no Slack/Resend/Claude keys), decrypted on the controller by the `community.sops` lookup |
| Stack on the host | `/opt/groupscout` (Compose files, config, `.env` mode 0600), run rootless as `deploy` (uid 2000) |
| Start at boot | lingering for `deploy`, plus the user unit `groupscout.service` (`podman compose up`) |

The **controller** is your dev WSL distro (`Ubuntu`). It runs ansible-core 2.21.4 and ansible-lint 26.9.0 from `uv tool` on Python 3.12, and holds the age key for SOPS. WSL 2 distros share one network namespace, so the controller reaches the host's sshd at `127.0.0.1:2222`.

## Create A Host

```powershell
# Windows PowerShell, from the backend checkout. The key pair is created once in the dev distro:
#   ssh-keygen -t ed25519 -N '' -C ansible@groupscout-prod -f ~/.ssh/groupscout_prod_ed25519
powershell -ExecutionPolicy Bypass -File deploy\windows\new-wsl-host.ps1 `
    -PublicKeyFile \\wsl.localhost\Ubuntu\home\<you>\.ssh\groupscout_prod_ed25519.pub
```

The script:
- installs Ubuntu 26.04 as `groupscout-prod`;
- turns on systemd and turns off Windows interop;
- creates `deploy` with uid 2000, subordinate ids 300000–365535 and passwordless sudo;
- starts sshd on `127.0.0.1:2222`, key only.

It's safe to re-run: an existing distro gets the same settings again, and a `deploy` user on another uid is moved to 2000.

## Configure It And Start The Stack

```sh
# In the dev distro, once per machine:
uv tool install --python 3.12 ansible-core==2.21.4
uv tool install --python 3.12 ansible-lint==26.9.0 --with ansible-core==2.21.4

cd /mnt/c/Users/alvin/GolandProjects/groupscout
make ansible-deps    # pinned collections
make ansible-apply   # configure and start; a second run reports changed=0
make ansible-check   # drift check (--check --diff)
make ansible-lint
```

**Keep the distro running while Ansible runs.** WSL stops a distro about 15 seconds after its last `wsl.exe` session ends, even with services running. Hold one open, for example `wsl.exe -d groupscout-prod -u root --exec sleep 3600` in another terminal. The SSH connection alone doesn't count.

The Makefile sets `ANSIBLE_CONFIG`, because Ansible ignores an `ansible.cfg` in a world-writable directory, and every path under `/mnt/c` is one in WSL.

## WSL Gotchas (All Handled)

| Problem | Effect | Fix |
|---|---|---|
| Every WSL 2 distro shares one **binfmt_misc** table. systemd ≥ 256 (Ubuntu 26.04 ships 259) flushes it at shutdown whenever it is mounted (`disable_binfmt`). | Stopping `groupscout-prod` deleted `WSLInterop`, so `cmd.exe`, `wsl.exe` and `podman.exe` failed with "Exec format error" in **every** distro. | The host doesn't use interop. `wsl.conf` sets `[interop] enabled=false`; the binfmt units are masked; `wsl-binfmt-detach.service` unmounts `binfmt_misc` at boot, so shutdown has nothing to flush. Verified across terminate and idle-stop cycles. |
| Distros share one **cgroup tree**. | A second `user@1000.service` can't start: "Failed to spawn executor: Device or resource busy". The Podman machine's user is uid 1000; so is your dev user, whose `user@1000` has been failing all along, which is why the dev distro reports "degraded". | `deploy` is uid 2000 with its own subordinate id range. `site.yml` refuses uid 1000 on WSL. |
| WSL **idle-stops** a distro without an attached `wsl.exe` session. | Services don't keep it alive. | A keep-alive session (above). At cutover, the Windows logon task holds one permanently. |
| Distros share **ports**. | The live stack holds 8080, 5678, 3000 and 5432. | Staging uses 18080, 15678 and 13000 through `GROUPSCOUT_*_HOST_PORT`. |
| `podman-restart.service` only restarts `restart: always` containers. | Compose uses `unless-stopped`, so nothing came back after a restart. | `groupscout.service` runs `podman compose up` at boot. Verified: the stack was healthy 15 s after a cold start. |
| A UFW default-deny would apply to the shared network namespace. | It would block traffic for every distro, the Podman machine included. | No UFW on WSL hosts; WSL's NAT keeps the LAN out. Other hosts get UFW. |

**If interop breaks anyway** ("Exec format error"), run this in the dev distro:

```sh
echo ':WSLInterop:M::MZ::/init:PF' | sudo tee /proc/sys/fs/binfmt_misc/register
```

## Cutover From The Podman Machine

Tracked in `groupscout-site-6ah.18`. Do it away from the Sun/Tue/Thu 18:00 cadence.

**Prerequisites:**
- Merge #15 (Phase 9), #16 and #17. Set `groupscout_image_tag` in `group_vars/all/main.yml` to the `main` SHA that includes Phase 9.
- Raise WSL's memory. `C:\Users\alvin\.wslconfig` caps all distros at 4 GB; set `memory=12GB`, then `wsl --shutdown`. That restarts every distro; the Podman autostart task brings the live stack back.
- Write the Windows logon task that starts `groupscout-prod` and holds its keep-alive, replacing the Podman machine autostart.

**Steps:**
1. Back up the live database age-encrypted, the way Phase 3 did.
2. Stop writes on the old stack: `podman stop groupscout_n8n groupscout_app` on the Podman machine.
3. Remove the staging stack and its volumes on `groupscout-prod` (`podman compose ... down --volumes`). The Postgres image only reads its password on first start, so the staging volume can't be reused.
4. Delete the four staging settings from `host_vars/groupscout-prod.yml`, then start Postgres alone with the production secrets: `make ansible-apply ANSIBLE_ARGS="-e '{\"groupscout_services\":[\"postgres\"]}'"`.
5. Restore the dump into it: `pg_dump` on the old stack piped to `psql` on the new one.
6. Stop the rest of the old stack, which frees ports 8080, 5678 and 3000.
7. Run `make ansible-apply`. The full stack comes up on the standard ports. n8n's bootstrap imports and publishes the cadence workflow into its fresh volume, and the backend's idempotency keys prevent a double delivery.
8. Verify:
   - `/health` reports the database OK and the right version, and leads and raw inputs are intact.
   - The n8n cadence workflow is active.
   - Grafana rules are healthy.
   - After the next run, the Healthchecks check is up.
9. Switch the Windows logon task to the WSL host. Keep the Podman machine for development only.

**Rollback:** start the old containers on the Podman machine again (`podman start groupscout_postgres groupscout_app groupscout_n8n`) and stop `groupscout-prod`.

## Related

- Plan: [INFRASTRUCTURE_AS_CODE.md](../planning/INFRASTRUCTURE_AS_CODE.md) Phase 5, D1 and D4
- Windows autostart and the dev Podman machine: [PODMAN_MIGRATION.md](./PODMAN_MIGRATION.md)
- Secrets: [SECRETS.md](./SECRETS.md) · Monitoring: [MONITORING.md](./MONITORING.md) · CI: [CI.md](./CI.md)
