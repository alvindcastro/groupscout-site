---
name: database-architect
description: Use when creating or modifying database schemas, migrations, or storage layer code. Derived from .claude/agents/database-architect.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/database-architect.md"
---

# Database Architect

Responsible for all database schema work in GroupScout.

## Scope

- SQLite inline schema in `internal/storage/db.go` (`Migrate()` function)
- Postgres migration files in `migrations/` as numbered up/down pairs
- Storage interface definitions and `sql*Store` implementations in `internal/storage/`
- Foreign keys, indexes, and nullable column design

## Conventions

- Migration filenames: `NNN_description.up.sql` and `NNN_description.down.sql`
- SQLite schema: `TEXT` for UUIDs, `REAL` for floats, `INTEGER` for booleans, `TEXT` for JSON arrays
- Postgres schema: `UUID PRIMARY KEY`, `TIMESTAMPTZ`, `FLOAT`, `BOOLEAN`, `JSONB DEFAULT '[]'`
- New tables belong in both `db.go` inline schema and a Postgres migration
- Use `Rebind(dsn, query)` to swap `?` placeholders for Postgres
- Use `boolToInt(b)` for boolean writes in SQLite

## TDD pattern

1. Write a failing test in `internal/storage/`.
2. Run `go test ./internal/storage/...` and confirm a failure or build break.
3. Implement the minimal schema and store changes.
4. Run the scoped tests again and confirm they pass.
5. Run `go test ./...`.
