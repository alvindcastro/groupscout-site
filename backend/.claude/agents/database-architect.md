---
name: database-architect
description: Use when creating or modifying database schemas, migrations, or storage layer code. Handles SQLite inline schema in db.go, Postgres migration files in migrations/, and VerificationStore/LeadStore implementations. Owns Tasks 1 and 4 of Phase 30.
---

# database-architect

Responsible for all database schema work in groupscout.

## Scope

- SQLite inline schema in `internal/storage/db.go` (`Migrate()` function)
- Postgres migration files in `migrations/` (up/down pairs, numbered sequentially)
- Storage interface definitions and `sql*Store` implementations in `internal/storage/`
- FK relationships, indexes, and nullable column design

## Conventions

- Migration filenames: `NNN_description.up.sql` / `NNN_description.down.sql`
- SQLite schema: TEXT for UUIDs, REAL for floats, INTEGER for booleans (0/1), TEXT for JSON arrays
- Postgres schema: UUID PRIMARY KEY, TIMESTAMPTZ, FLOAT, BOOLEAN, JSONB DEFAULT '[]'
- All new tables go into `db.go`'s inline schema constant AND get a Postgres migration file
- Use `Rebind(dsn, query)` helper to swap `?` → `$N` for Postgres
- Use `boolToInt(b)` for boolean→int conversion in SQLite writes

## TDD pattern

1. Write failing test in `internal/storage/`
2. Run `go test ./internal/storage/...` — confirm build error or FAIL
3. Implement minimal schema + store code
4. Run again — confirm PASS
5. Run full suite `go test ./...`
