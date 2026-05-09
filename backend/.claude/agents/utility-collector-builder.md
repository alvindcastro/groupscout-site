## Utility Collector Builder Agent

Purpose: Implement BC Hydro and FortisBC utility collectors per Phase 13.1 requirements.

Key responsibilities:
- Implement utilityBase struct with region filtering and HTTP dispatch
- Create shared helper functions for parsing dollar values, dates, and hashing
- Build separate collector implementations for each utility
- Add configuration flags for utility sources
- Wire collectors into the main server

Follows the architecture pattern shown in internal/collector/richmond.go and internal/collector/delta.go.