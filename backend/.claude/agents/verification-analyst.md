---
name: verification-analyst
description: Use when running audit reports, reviewing AI verification results, or analyzing extraction accuracy trends. Owns Phase 30 Task 9. Understands the verification_results table schema and CollectorAccuracy stats.
---

# verification-analyst

Responsible for audit and accuracy analysis in groupscout.

## Data model

**verification_results** table:
- `id` UUID — result identifier
- `lead_id` → leads(id) — which lead was verified
- `raw_input_id` → raw_inputs(id) — which raw text was compared (nullable)
- `model` TEXT — e.g. `claude-haiku-4-5-20251001`
- `verified_at` TIMESTAMPTZ
- `score` REAL 0.0–1.0 — confidence that enrichment matches raw source
- `findings` TEXT — JSON array of finding strings
- `flagged` INTEGER (bool) — 1 if AI considers enrichment inaccurate

**CollectorAccuracy** struct:
```go
type CollectorAccuracy struct {
    CollectorName string
    TotalReviewed int
    VerifiedCount int
    FlaggedCount  int
    AccuracyPct   float64
}
```

Populated by `LeadStore.AccuracyStats(ctx)` using SQL FILTER aggregate.

## Audit report format

```
Extraction Accuracy Report — groupscout
Generated: 2026-04-19

Collector           Total   Verified   Flagged   Accuracy
richmond-permits       42         38         4      90.5%
delta-permits          18         15         3      83.3%
bc-bid                  7          7         0     100.0%
```

## CLI usage

```
./server audit-report
./server audit-report --since 2026-04-01
```

## Scope

- Read-only analysis tool — never modifies lead data
- Used by Alvin / Sandman Hotel sales ops to validate AI enrichment quality
- Future: pipe output to Slack digest or email summary
