# Secrets As Code (SOPS + age)

IaC Phase 3 (`groupscout-site-6ah.4`). The runtime secrets live in the backend repo, encrypted with [SOPS](https://github.com/getsops/sops) to [age](https://github.com/FiloSottile/age) keys. A fresh clone plus an age private key can render a working `.env`.

**Status:** done on 2026-10-14 and merged to backend `main` as `a9f8323`.
- The live home stack runs from a `.env` rendered out of `deploy/secrets/prod.sops.env`. A fresh GitHub clone plus the age key renders a byte-identical `.env`.
- The Postgres role password, Grafana admin password and `API_TOKEN` were rotated in place. `N8N_ENCRYPTION_KEY` was copied from the `n8n_data` volume.
- The pre-rotation `.env` is kept age-encrypted at `C:\Users\alvin\groupscout-backups\2026-10-14-iac-phase3\env-before-rotation.age`. Decrypt it with `age -d -i ~/.config/sops/age/keys.txt`.
- The age key backup is still open in `groupscout-site-6ah.13`.

## What Lives Where

| Item | Location | In git? |
|---|---|---|
| Encrypted Compose environment | `deploy/secrets/prod.sops.env` (dotenv; key names readable, values and comments encrypted) | Yes |
| Recipients and creation rules | `.sops.yaml` | Yes |
| Rendered plaintext | `.env` in the checkout (mode `0600` on Linux) | No (`.gitignore`) |
| Staging environment | `deploy/secrets/staging.sops.env`: throwaway values for the staging stack on the WSL host, with no Slack, Resend or Claude keys (Phase 5) | Yes |
| Rendered plaintext on the WSL host | `/opt/groupscout/.env` on `groupscout-prod` (mode `0600`). Ansible decrypts on the controller with the `community.sops` lookup (`no_log`), so the age key never leaves the dev distro ([WSL_HOST.md](./WSL_HOST.md)) | No |
| Ansible SSH key | `~/.ssh/groupscout_prod_ed25519` in the dev distro; the only key `deploy` accepts on `groupscout-prod` | **Never** |
| age private key | `~/.config/sops/age/keys.txt` in WSL (`%AppData%\sops\age\keys.txt` for Windows-native sops) | **Never** |
| Template for setups without the key | `.env.example` | Yes |

All three GitHub repos are **public**, so the encrypted file is public too. That is the normal SOPS model: the ciphertext is safe while the age private key stays private. If the key ever leaks, every value that was ever in the file must be rotated, because old commits stay decryptable with the old key.

## One-Time Setup

Install the pinned tools. These are the versions Phase 3 was built with; each download is checked against the release checksum.

```sh
# WSL / Linux (amd64), into ~/.local/bin
cd "$(mktemp -d)"
gh release download v3.13.3 -R getsops/sops -p 'sops-v3.13.3.linux.amd64' -p 'sops-v3.13.3.checksums.txt'
grep ' sops-v3.13.3.linux.amd64$' sops-v3.13.3.checksums.txt | sha256sum -c -
gh release download v1.3.2 -R FiloSottile/age -p 'age-v1.3.2-linux-amd64.tar.gz'
echo "cbe24006683f8eb669266162894b9a522a1af52f2665fbc63a4bb032ed26ac10  age-v1.3.2-linux-amd64.tar.gz" | sha256sum -c -
gh release download v8.30.1 -R gitleaks/gitleaks -p 'gitleaks_8.30.1_linux_x64.tar.gz' -p 'gitleaks_8.30.1_checksums.txt'
grep ' gitleaks_8.30.1_linux_x64.tar.gz$' gitleaks_8.30.1_checksums.txt | sha256sum -c -
tar -xzf age-v1.3.2-linux-amd64.tar.gz && tar -xzf gitleaks_8.30.1_linux_x64.tar.gz gitleaks
install -m 755 sops-v3.13.3.linux.amd64 ~/.local/bin/sops
install -m 755 age/age age/age-keygen gitleaks ~/.local/bin/
```

The workstation key was created on 2026-10-14 with `age-keygen -o ~/.config/sops/age/keys.txt` (mode `0600`). Its public key is the `alvin_wsl` entry in `.sops.yaml`.

### Back Up The Key (Do This Once)

The private key is the root of trust. Without it, nobody can decrypt `prod.sops.env`.

1. Copy the whole content of `~/.config/sops/age/keys.txt` (three lines, including `AGE-SECRET-KEY-1…`) into a password-manager entry named "GroupScout age key".
2. Keep one offline copy, such as a printout or a USB drive kept away from the PC.
3. Run the recovery test below from the password-manager copy, not from the WSL file. Tracked as `groupscout-site-6ah.13`.

### Recovery Test

```sh
cd "$(mktemp -d)"
git clone https://github.com/alvindcastro/groupscout.git && cd groupscout
# paste the backed-up key into a temp file
(umask 077; cat > /tmp/age-recovery.txt)   # paste, then Ctrl-D
SOPS_AGE_KEY_FILE=/tmp/age-recovery.txt SECRETS_OUT=/tmp/recovered.env make secrets-render
cmp /tmp/recovered.env /mnt/c/Users/alvin/GolandProjects/groupscout/.env && echo "backup key works"
rm -f /tmp/age-recovery.txt /tmp/recovered.env
```

## Daily Use

All targets wrap `deploy/scripts/secrets.sh`. Without `make` (Git Bash on Windows), run `sh deploy/scripts/secrets.sh <command>` instead. That needs `sops` on `PATH` and the key in `%AppData%\sops\age\keys.txt`.

| Command | What it does |
|---|---|
| `make secrets-edit` | Opens the decrypted file in `$EDITOR`. It re-encrypts on save, and closing without changes leaves the file alone. |
| `make secrets-render` | Writes `.env` from the encrypted file, using mode `0600` and an atomic rename. It refuses to replace a `.env` that differs, and lists which keys differ. |
| `make secrets-render FORCE=1` | Overwrites a `.env` that differs from the encrypted file. |
| `make secrets-diff` | Names the keys that differ between `.env` and the encrypted file. It never prints values. |
| `make secrets-updatekeys` | Re-encrypts every `deploy/secrets/*.sops.env` to the recipients now listed in `.sops.yaml`. |
| `make secrets-scan` | Runs gitleaks over the full history and checks the files described in [Secret Scanning](#secret-scanning). |

`SECRETS_ENV=staging make secrets-edit` creates or edits `deploy/secrets/staging.sops.env` with the same rules. `SECRETS_OUT=path` renders somewhere other than `.env`.

To change a value:

1. Run `make secrets-edit`, change the value, then save and quit.
2. Run `make secrets-render FORCE=1`.
3. Recreate the services that read it, for example `podman compose up -d groupscout n8n`.
4. Commit `deploy/secrets/prod.sops.env`.

The git diff shows which keys changed, never their values.

If you edited `.env` by hand, `make secrets-diff` lists the drift. Copy those changes into `make secrets-edit`, then render again.

## Adding A Recipient (New Machine, Prod Host, CI)

1. On the new machine, run `age-keygen -o ~/.config/sops/age/keys.txt` and copy the `age1…` public key it prints.
2. In `.sops.yaml`, add it under `keys:` with an anchor, and reference it in each `creation_rules` key group.
3. On a machine that can already decrypt, run `make secrets-updatekeys`, then commit `.sops.yaml` and the re-encrypted files.

Each age key in the group can decrypt on its own. To remove a recipient, delete it, run `make secrets-updatekeys`, and **rotate the values**: the removed key can still decrypt every older commit.

## Rotating A Secret

Note when each change takes effect. Several images read their secret only when the volume is first created, so changing `.env` alone does nothing for an existing stack.

| Secret | Rotation |
|---|---|
| `POSTGRES_PASSWORD` | The Postgres image applies it only on first init. Edit both `POSTGRES_PASSWORD` and the password inside `DATABASE_URL` (used by host-side `go run`). Then run `printf "ALTER ROLE groupscout WITH PASSWORD '%s';\n" "$NEW" \| podman exec -i groupscout_postgres psql -U groupscout -d groupscout` (local socket, trust auth, password not on the command line). Finally render and recreate `postgres` and `groupscout`. |
| `GRAFANA_ADMIN_PASSWORD` | Grafana applies it only on first init. Edit, render, recreate `grafana`, then run `printf '%s\n' "$NEW" \| podman exec -i groupscout_grafana grafana cli admin reset-admin-password --password-from-stdin`. |
| `API_TOKEN` | Edit, render, and recreate `groupscout` and `n8n`. n8n receives it as `GROUPSCOUT_API_TOKEN`. Update any Bruno or curl environment that holds the old token. |
| `N8N_ENCRYPTION_KEY` | **Do not rotate casually.** It must equal the key stored in the `n8n_data` volume, or n8n refuses to start ("Mismatching encryption keys"), and it encrypts n8n credentials. The encrypted file keeps a copy so a volume restore stays usable. |
| `CLAUDE_API_KEY`, `SLACK_*`, `OPS_SLACK_WEBHOOK_URL`, `RESEND_API_KEY` | Rotate at the provider, then edit, render, and recreate the consumers listed in IaC plan Appendix A. |
| `CADENCE_HEARTBEAT_URL`, `HEALTHCHECKS_API_KEY` | Regenerate the ping key or API key in the Healthchecks.io project settings. Edit, render, and recreate `groupscout`; the API key is only read by `make monitors-apply`. See [MONITORING.md](./MONITORING.md). |

Generate URL-safe values with `openssl rand -hex 24` (or `-hex 32` for `API_TOKEN`). `POSTGRES_PASSWORD` sits inside a URL, so avoid `@ : / ?`.

Before a risky rotation, keep an encrypted copy of the current `.env` rather than a plaintext one: `age -r "$(age-keygen -y ~/.config/sops/age/keys.txt)" -o env-before.age .env`.

## Secret Scanning

- **CI.** `.github/workflows/secret-scan.yml` runs on every push and pull request, on `ubuntu-24.04`. It checks out with `actions/checkout` pinned to a SHA and the full history. It installs gitleaks 8.30.1, checksum-verified, then runs `make secrets-scan`. Output is redacted because Actions logs on a public repo are public.
- **What `secrets-scan` fails on:**
  - any gitleaks finding not listed in `.gitleaksignore`;
  - a tracked plaintext env or key file: `*.env`, `.env.*` (other than `.env.example`), `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.jks`, `*.tfstate`, `id_rsa`/`id_ecdsa`/`id_ed25519`, or `keys.txt`;
  - a `*.sops.env` that is not fully SOPS-encrypted.
- **`.gitleaksignore`** lists the 43 reviewed findings in published history, each identified by fingerprint. None is a real credential: they are placeholder curl tokens such as `YOUR_API_TOKEN` and `dev-token`, a deliberately planted fake key in `data/evals/groupscout/adversarial_cases.jsonl`, and an `appXXXX` Airtable id. Fix and rotate new findings; don't add them to this list.
- **History review on 2026-10-14.** All three repos (backend, UI, coordination) were scanned with `--log-opts=--all`, giving 43, 13 and 548 findings. Every one was a placeholder, a test fixture, or a Beads memory key name. Phase 4 adds the same CI scan to the UI repo, with its 13 reviewed findings in the UI `.gitleaksignore` (UI PR #1, merged into `task/production-ui-compose`). The coordination repo has no CI scan yet (`groupscout-site-ymx`).
- **GitHub.** Secret scanning and push protection are enabled on all three repos.
- **Smoke test.** `internal/smoke/secrets_test.go` checks the `.sops.yaml` rules, that everything under `deploy/secrets/` is encrypted, that `prod.sops.env` has every variable `compose.prod.yml` requires, and the CI wiring.

## Related

- Integration tests delete rows, so run them only against a throwaway database ([TESTING.md](./TESTING.md), section 4). Code guards against pointing them at the live database are tracked in `groupscout-site-sfi` (P1).
- CI, image publishing and Renovate: [CI.md](./CI.md)
- Secrets inventory: [INFRASTRUCTURE_AS_CODE.md Appendix A](../planning/INFRASTRUCTURE_AS_CODE.md#appendix-a-secrets-inventory)
- n8n encryption key behaviour: [DOCKER.md](./DOCKER.md) and [N8N_GUIDE.md](./N8N_GUIDE.md)
