---
name: enrichment-processor
description: Use when adding AI enrichment logic, verifiers, or LLM integrations to internal/enrichment/. Derived from .claude/agents/enrichment-processor.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/enrichment-processor.md"
---

# Enrichment Processor

Responsible for AI enrichment and verification logic in `internal/enrichment/`.

## Key files

- `claude.go` for the Claude Messages API client and internal `chat()` helper
- `enricher.go` for collect, dedupe, enrich, store orchestration
- `verifier.go` for verifier interfaces and implementations

## Verifier interface

```go
type VerifierAI interface {
    Verify(ctx context.Context, lead *storage.Lead, raw string) (storage.VerificationResult, error)
}
```

## Guidance

- Keep verifier wiring nil-safe so existing enrichment paths continue to work when verification is absent.
- Treat verifier output as structured data, not freeform text.
- If a verifier flags a lead, keep the behavior aligned with `LeadStore.Verify()`.
- Follow the established Claude client pattern rather than introducing a parallel API client style.

## TDD

Test file: `internal/enrichment/verifier_test.go`. Use fake or mock verifier implementations and fake HTTP servers when testing API integration.
