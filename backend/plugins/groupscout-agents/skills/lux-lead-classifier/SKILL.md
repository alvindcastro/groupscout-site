---
name: lux-lead-classifier
description: Use when classifying inbound LUX construction leads before sequence generation. Derived from .claude/agents/lux-lead-classifier.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/lux-lead-classifier.md"
---

# LUX Lead Classifier

Classify inbound LUX construction leads as the first step in the MVP-B follow-up pipeline. The original `docs/mvps/mvp-b/` files are historical/migrated in this coordinator docs tree; use this skill and the backend source repo's `.claude/agents/lux-lead-classifier.md` as the current role reference.

## Historical source files

- `docs/mvps/mvp-b/payload.json`
- `docs/mvps/mvp-b/payload_alt.json`
- `docs/mvps/mvp-b/prompts/user_classify.txt`
- `docs/mvps/mvp-b/prompts/system_brand_voice.txt`

## Required output

Return exactly this JSON shape:

```json
{
  "lead_tier": "high",
  "project_category": "commercial_renovation",
  "key_detail": "two tenant spaces in our office building downtown",
  "urgency_signal": false
}
```

Do not add preamble or markdown.

## Classification rules

- `lead_tier`: `high`, `medium`, or `low`
- `project_category`: `commercial_renovation`, `multi_family`, `custom_home`, `addition_or_remodel`, or `unknown`
- `key_detail`: the most specific detail from the lead in up to 15 words, or `null`
- `urgency_signal`: true only for explicit timing pressure or named dates

## When tuning

- Update the active migrated prompt source or this skill's rules, depending on where the workflow now keeps prompts
- Remind the user to sync changes to the n8n workflow copy
- Test against both sample payloads
