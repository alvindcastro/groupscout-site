---
name: enrichment-processor
description: Use when adding AI enrichment logic, verifiers, or LLM integrations to internal/enrichment/. Owns Phase 30 Task 6 (AI Verifier). Follows the existing claude.go chat() pattern and VerifierAI interface.
---

# enrichment-processor

Responsible for AI enrichment and verification logic in `internal/enrichment/`.

## Key files

- `claude.go` — Claude Messages API client; unexported `chat()` method
- `enricher.go` — Orchestrates collect→dedup→enrich→store pipeline; has optional `Verifier` hook
- `verifier.go` — (Task 6) VerifierAI interface + ClaudeVerifier implementation

## VerifierAI interface

```go
type VerifierAI interface {
    Verify(ctx context.Context, lead *storage.Lead, raw string) (storage.VerificationResult, error)
}
```

## ClaudeVerifier

- Uses `claude-haiku-4-5-20251001` model (fast, cheap)
- Prompt: compare enriched lead fields against raw source text; return JSON with `score`, `findings[]`, `flagged`
- If `Flagged=true`, auto-sets `lead.VerificationStatus = "flagged"` via `LeadStore.Verify()`

## Enricher hook

```go
type Enricher struct {
    // ...
    Verifier VerifierAI // optional; nil = skip verification
}
```

Verifier is called after successful enrichment and storage. Nil-safe — existing tests must not break.

## TDD

Test file: `internal/enrichment/verifier_test.go`. Use a `mockVerifier` or `fakeVerifier` for handler tests. The Claude API call is tested via a fake HTTP server or by injecting a mock `VerifierAI`.
