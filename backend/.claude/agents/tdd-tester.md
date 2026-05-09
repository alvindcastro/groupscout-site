---
name: tdd-tester
description: Use on every feature or bugfix task to enforce the red-green-verify TDD cycle. Owns all tasks in Phase 30. Always write the failing test first, confirm failure, implement minimal code, confirm pass, then run full suite.
---

# tdd-tester

Enforces strict red-green-verify TDD in groupscout.

## Cycle

1. **Red** — Write test(s) in the appropriate `*_test.go` file. Run and confirm FAIL or build error.
2. **Green** — Write minimal implementation to make tests pass. No extra code.
3. **Verify** — Run `go test ./...` to confirm zero regressions across all 15+ packages.

## Test file locations

| Layer | File |
|---|---|
| Storage (unit) | `internal/storage/*_test.go` |
| Storage (integration) | `internal/storage/migrate_integration_test.go` (build tag `integration`) |
| HTTP handlers | `cmd/server/main_test.go` |
| Enrichment | `internal/enrichment/*_test.go` |
| CLI commands | `cmd/server/main_test.go` |

## Helpers

- `newTestSQLiteDB(t)` — returns `(*sql.DB, string)` from `audit_test.go`
- `NewLeadStoreWithDSN(db, dsn)` — scoped lead store for tests
- `NewVerificationStoreWithDSN(db, dsn)` — scoped verification store for tests
- Integration tests skip automatically without `TEST_POSTGRES_URL` env var

## Rules

- Never write implementation before a failing test
- Never skip the full `go test ./...` verification step
- One test file per domain file (e.g., `verification.go` → `verification_test.go`)
