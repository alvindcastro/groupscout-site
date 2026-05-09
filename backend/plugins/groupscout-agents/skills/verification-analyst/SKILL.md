---
name: verification-analyst
description: Use when running audit reports, reviewing AI verification results, or analyzing extraction accuracy trends. Derived from .claude/agents/verification-analyst.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/verification-analyst.md"
---

# Verification Analyst

Analyze verification output, audit reports, and extraction accuracy trends in GroupScout.

## Data model

`verification_results` contains:

- `id`
- `lead_id`
- `raw_input_id`
- `model`
- `verified_at`
- `score`
- `findings`
- `flagged`

`CollectorAccuracy` contains:

```go
type CollectorAccuracy struct {
    CollectorName string
    TotalReviewed int
    VerifiedCount int
    FlaggedCount  int
    AccuracyPct   float64
}
```

## CLI usage

```bash
./server audit-report
./server audit-report --since 2026-04-01
```

## Rules

- Read-only analysis only
- Never modify lead data
- Keep reporting aligned with the existing audit-report output format
