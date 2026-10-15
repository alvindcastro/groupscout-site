# CI And Image Publishing

IaC Phase 4 (`groupscout-site-6ah.5`). GitHub Actions tests every pull request in the backend and UI repos. A commit merged to `main` gets a container image in GitHub Container Registry (GHCR), tagged with its full commit SHA.

**Status:** the backend has published images since 2026-10-15. The UI image waits for UI `main`.
- **Backend:** PR [#14](https://github.com/alvindcastro/groupscout/pull/14) was squash-merged to `main` as `4ec5252`.
  - The first `main` run passed every job and pushed `ghcr.io/alvindcastro/groupscout:4ec52525542e29ed561420c5a1550232a0d1ae18` and `:main` (digest `sha256:5b2f62a2ba9c…`).
  - The package is already public: the registry serves the manifest without a login.
  - An anonymous `podman pull` of the SHA tag, run against a throwaway Postgres, answered `/health` with the database OK and that version.
- **UI:** PR [#1](https://github.com/alvindcastro/groupscout-ui/pull/1) merged into `task/production-ui-compose` as `758fc56`. UI images publish once that branch reaches UI `main`.
- Branch protection and the Renovate app are still to do (see [Owner Steps](#owner-steps)).

## What Runs

| Repo | Workflow | Job | What it checks | Runs on |
|---|---|---|---|---|
| backend | `ci.yml` | `go` | `gofmt -l`, `go vet ./...`, `go test ./...`, `make eval-gate` | PRs, pushes to `main` |
| backend | `ci.yml` | `integration` | `go test -tags integration -p 1 ./...` against a throwaway pgvector Postgres service container | PRs, pushes to `main` |
| backend | `ci.yml` | `compose` | `make compose-check`: every dev profile set plus the prod overlay render | PRs, pushes to `main` |
| backend | `ci.yml` | `image` | Builds the image, starts it with the prod overlay, checks `/health` reports the database OK and this commit's SHA, then runs the vulnerability gate. On `main` it pushes. | After the three jobs above pass |
| backend | `ci.yml` | `ansible` | `ansible-lint` (production profile) and `ansible-playbook --syntax-check` of `deploy/ansible`, with pinned ansible-core 2.21.4, ansible-lint 26.9.0 and collections. Added in Phase 5 (PR #17). | PRs, pushes to `main` |
| backend | `secret-scan.yml` | `scan` | `make secrets-scan` (gitleaks over the full history, redacted) | Every push and PR |
| UI | `ci.yml` | `test` | `npm test` inside `node:22.23.3-trixie-slim`, the Dockerfile's base image | PRs, pushes to `main` |
| UI | `ci.yml` | `image` | Builds `--target production`, checks `/healthz` and the `node` user, then runs the vulnerability gate. On `main` it pushes. | After `test` passes |
| UI | `secret-scan.yml` | `scan` | gitleaks over the full history (redacted), plus a check that no `.env` or key file is tracked | Every push and PR |

The plan listed a separate `images.yml`. Publishing lives in the `image` job of `ci.yml` instead, so an image is pushed only after that commit's tests pass.

## Images

| Image | Tags | Built from |
|---|---|---|
| `ghcr.io/alvindcastro/groupscout` | `<full 40-char SHA>`, `main` | backend `Dockerfile`; `VERSION` = the full SHA, which `/health` reports |
| `ghcr.io/alvindcastro/groupscout-ui` | `<full 40-char SHA>`, `main` | UI `Dockerfile`, `production` target |

- Deploy by SHA, never by `main`. The prod overlay already requires `GROUPSCOUT_IMAGE` and `GROUPSCOUT_IMAGE_TAG`:
  ```sh
  GROUPSCOUT_IMAGE=ghcr.io/alvindcastro/groupscout GROUPSCOUT_IMAGE_TAG=<sha> \
    docker compose -f docker-compose.yml -f deploy/compose/compose.prod.yml up -d
  ```
- Images are `linux/amd64` only. Add arm64 if D1 picks an ARM host.
- Local builds still report the short SHA (`make docker-up`). CI images report the full SHA.
- Re-running a `main` workflow rebuilds and overwrites that SHA's tag. The content is the same commit, but the image digest changes.

## Vulnerability Gate

The `image` jobs run [grype](https://github.com/anchore/grype) 0.119.0, downloaded and checked against its release checksum:

```sh
grype docker:<image> --only-fixed --fail-on critical
```

- The log lists every finding that has a fix, but only a **fixable critical** finding fails the job. Findings with no fix don't count.
- Grype rather than Trivy: Trivy's release tags were hijacked in a supply-chain attack in 2026, and grype installs as one checksum-verified binary, like gitleaks.
- What the gate found on 2026-10-15:
  - Backend: no fixable criticals. There are 6 fixable Highs: `golang.org/x/net` 0.52, `golang.org/x/text` 0.35, and Alpine `zlib` 1.3.2-r0. Renovate's grouped Go-modules and image PRs pick these up.
  - UI: `node:22.23.3-bookworm-slim` carried 5 fixable critical `perl-base` CVEs, so the UI moved to `node:22.23.3-trixie-slim`, which has none. The remaining Highs are in the npm copy bundled with Node (`groupscout-site-6ah.15`).

Reproduce the scan locally against a Podman image:

```sh
podman save --format docker-archive <image> > /tmp/image.tar
grype docker-archive:/tmp/image.tar --only-fixed --fail-on critical
```

When the gate fails, update the base image tag or digest, or the dependency, to the fixed version. Don't add an ignore rule.

## Supply Chain

- Every third-party action is pinned to a commit SHA, with the version in a comment. The `ci_test.go` and `ci-contract.test.js` guard tests fail on an unpinned `uses:`.
- New pins use releases at least two weeks old, so a hijacked fresh release has time to be noticed:

  | Tool | Version | Pin |
  |---|---|---|
  | `actions/checkout` | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
  | `actions/setup-go` | v7.0.0 | `b7ad1dad31e06c5925ef5d2fc7ad053ef454303e` |
  | grype | 0.119.0 | sha256 `3fa2dc4b924621ab65404cf08d0b8438d896d80ab949c9d5a4ca283c36004c9b` |
  | gitleaks | 8.30.1 | sha256 `551f6fc83ea457d62a0d98237cbad105af8d557003051f41f3e7ca7b3f2470eb` |
  | ansible-core / ansible-lint | 2.21.4 / 26.9.0 | pip versions in a venv; collections pinned in `deploy/ansible/requirements.yml` |

- Renovate doesn't bump the grype and gitleaks downloads. Update `*_VERSION` and `*_SHA256` together from the release's checksums file.
- `GITHUB_TOKEN` is read-only except in the `image` job, which gets `packages: write`. The push step runs only for non-PR events on `main` in the owner's repo, and passes the token to `docker login` on stdin.
- No workflow uses repository secrets, so pull requests from forks run with nothing to steal.

## Renovate

`renovate.json` in each repo:

- **Schedule:** before 6am Monday (America/Vancouver), at most 5 open PRs. Nothing automerges, and every PR has to pass CI.
- **Digest pinning:** the first run opens a "Pin dependencies" PR that adds `@sha256:` digests to Compose, Dockerfile and workflow images.
- **Cool-off:** new Go modules and Actions wait 7 days. Image digests don't, so base-image security rebuilds arrive the next Monday.
- **Groups:**
  - Backend: GitHub Actions, Go modules (minor and patch), and container images. n8n, Grafana and pgvector each get their own PR with a backup warning, because they migrate data forward and can't be downgraded. pgvector appears in Compose and the CI service, and both copies move in the same PR.
  - UI: GitHub Actions, and one Node image group that keeps the Dockerfile stages and the CI test container on the same image.
- **Not covered yet:** OpenTofu providers and Ansible collections. Renovate picks them up automatically when Phases 5 and 6 add them.

Validate a config change before pushing it:

```sh
npx --yes --package renovate@44.125.2 renovate-config-validator --strict
```

## Owner Steps

These need the repo owner on GitHub (`groupscout-site-6ah.14`):

1. ~~**Merge** backend PR #14, then UI PR #1.~~ Done on 2026-10-15. Claude Code's auto-mode guard blocks the agent from merging without review, so later PRs need the owner too.
2. **Branch protection on `main`** (backend, and UI once CI is on UI `main`):
   - Required checks: backend `go`, `integration`, `compose`, `ansible` (once PR #17 merges), `image`, `scan`; UI `test`, `image`, `scan`.
   - Settings: source GitHub Actions (app id 15368), "Require branches to be up to date" off, force pushes and deletion blocked.
   - "Include administrators" off, so the owner can still push an emergency fix.
3. **Install the Renovate GitHub app** (Mend Renovate, free) for `groupscout` and `groupscout-ui` only. Its onboarding PR shows the detected dependencies.
4. **Package visibility:** the backend package came out public, inheriting the public repo, so no step was needed. Check `groupscout-ui` the same way after its first `main` run:
   ```sh
   tok=$(curl -fsS 'https://ghcr.io/token?scope=repository:alvindcastro/groupscout-ui:pull' | jq -r .token)
   curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $tok" \
     -H 'Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json' \
     https://ghcr.io/v2/alvindcastro/groupscout-ui/manifests/main   # 200 = public
   ```
5. **UI `main`:** decide whether `task/production-ui-compose` (with the Phase 4 CI) merges into UI `main`. UI images publish only from there.

## Run CI Locally

```sh
# backend: go job
gofmt -l . ; go vet ./... && go test -count=1 ./... && make eval-gate
# backend: compose job (WSL without Docker: COMPOSE=<standalone docker-compose>)
make compose-check
# backend: integration job (throwaway database; see TESTING.md)
TEST_POSTGRES_URL='postgres://groupscout:test@localhost:55432/groupscout_test?sslmode=disable' \
  go test -tags integration -p 1 -count=1 ./...
# UI: test job
npm test
# backend: ansible job (controller tools: see WSL_HOST.md)
make ansible-deps && make ansible-lint
# workflow lint (rhysd/actionlint 1.7.12)
actionlint .github/workflows/*.yml
```

## Related

- Plan: [INFRASTRUCTURE_AS_CODE.md](../planning/INFRASTRUCTURE_AS_CODE.md) Phase 4
- Secret scanning: [SECRETS.md](./SECRETS.md#secret-scanning)
- Integration-test database: [TESTING.md](./TESTING.md)
- Compose files and the prod overlay: [DOCKER.md](./DOCKER.md)
