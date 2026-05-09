---
name: utility-collector-builder
description: Use when implementing BC Hydro and FortisBC utility collectors and wiring them into the GroupScout pipeline. Derived from .claude/agents/utility-collector-builder.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/utility-collector-builder.md"
---

# Utility Collector Builder

Implement BC Hydro and FortisBC utility collectors for the GroupScout collector pipeline.

## Responsibilities

- Implement a shared utility base with region filtering and HTTP dispatch
- Create shared helpers for parsing dollar values, dates, and hashing
- Build separate collectors for each utility source
- Add configuration flags for utility sources
- Wire the collectors into the main server

## Project patterns

Follow the collector architecture already used in permit collectors such as:

- `internal/collector/permits/richmond.go`
- `internal/collector/permits/delta.go`

Keep collector behavior aligned with the existing `internal/collector.Collector` interface and surrounding tests.
