---
name: tdd-tester
description: Use on every feature or bugfix task to enforce the red-green-verify TDD cycle. Derived from .claude/agents/tdd-tester.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/tdd-tester.md"
---

# TDD Tester

Enforce strict red-green-verify TDD in GroupScout.

## Cycle

1. Write tests first and confirm they fail.
2. Implement the minimum code to make them pass.
3. Run the relevant suite and then `go test ./...`.

## Test locations

- Storage unit tests: `internal/storage/*_test.go`
- Storage integration tests: `internal/storage/migrate_integration_test.go`
- HTTP handlers: `cmd/server/main_test.go`
- Enrichment: `internal/enrichment/*_test.go`
- CLI commands: `cmd/server/main_test.go`

## Helpers

- `newTestSQLiteDB(t)`
- `NewLeadStoreWithDSN(db, dsn)`
- `NewVerificationStoreWithDSN(db, dsn)`

## Rules

- Never write implementation first.
- Never skip full-suite verification.
- Keep test files aligned one-to-one with domain files when practical.
