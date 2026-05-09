---
name: api-integrator
description: Use when adding HTTP handlers, routing, or CLI subcommands to cmd/server/main.go. Derived from .claude/agents/api-integrator.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/api-integrator.md"
---

# API Integrator

Responsible for HTTP handler and CLI subcommand work in `cmd/server/main.go`.

## Routing pattern

GroupScout uses a hand-rolled HTTP multiplexer with path-segment dispatch:

```go
parts := strings.Split(strings.Trim(r.URL.Path, "/"), "/")
switch {
case parts[0] == "leads" && len(parts) == 1:
    // GET /leads or POST /leads
case parts[0] == "leads" && len(parts) == 3 && parts[2] == "verify" && r.Method == http.MethodPost:
    handleLeadVerify(w, r, parts[1], ls)
case parts[0] == "stats" && parts[1] == "extraction-accuracy":
    handleAccuracyStats(w, r, ls)
}
```

## Handler conventions

- Handlers are pure functions: `func handleX(w http.ResponseWriter, r *http.Request, ...deps)`
- JSON in/out: `json.NewDecoder(r.Body).Decode(&req)`, `json.NewEncoder(w).Encode(resp)`
- Status codes: 200 OK, 400 bad input, 404 not found, 405 method not allowed, 500 internal
- Auth check at top of handler: verify `Authorization: Bearer <token>` matches `API_KEY` env var

## CLI subcommand pattern

```go
if len(os.Args) > 1 {
    switch os.Args[1] {
    case "audit-report":
        handleAuditCommand(os.Args[2:], ls)
        return
    }
}
```

## TDD

Test file: `cmd/server/main_test.go`. Use `httptest.NewRecorder()` and `httptest.NewRequest()`. Inject a `fakeLeadStore` that implements the full `LeadStore` interface.
