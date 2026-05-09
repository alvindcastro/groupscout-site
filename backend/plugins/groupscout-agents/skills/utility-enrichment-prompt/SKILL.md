---
name: utility-enrichment-prompt
description: Use when defining or applying the structured extraction prompt for utility project signals. Derived from .claude/agents/utility-enrichment-prompt.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/utility-enrichment-prompt.md"
---

# Utility Enrichment Prompt

Extract structured data from BC Hydro and FortisBC utility project signals.

## Required fields

```json
{
  "general_contractor": "string or unknown",
  "project_type": "civil|commercial|industrial|utility|residential|unknown",
  "estimated_crew_size": 80,
  "estimated_duration_months": 6,
  "out_of_town_crew_likely": true,
  "priority_score": 9,
  "priority_reason": "Large civil project within 5km of hotel, groundbreaking imminent",
  "suggested_outreach_timing": "Reach out now - crews mobilizing in 4-6 weeks",
  "notes": "GC is PCL Construction. Check LinkedIn for travel coordinator."
}
```

## Special handling

- For BC Hydro, extract from tender documents and news announcements.
- For FortisBC, focus on infrastructure projects and service disruptions.
- Apply region filtering based on the relevant utility area.

Use this skill when designing prompt text, validating extraction outputs, or aligning utility signals with the downstream GroupScout schema.
