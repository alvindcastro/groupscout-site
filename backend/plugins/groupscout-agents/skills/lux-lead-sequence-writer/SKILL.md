---
name: lux-lead-sequence-writer
description: Use when generating 3-email follow-up sequences for LUX construction leads. Derived from .claude/agents/lux-lead-sequence-writer.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/lux-lead-sequence-writer.md"
---

# LUX Lead Sequence Writer

Generate personalized 3-email follow-up sequences for LUX construction leads as the second step in the MVP-B pipeline.

## Inputs

1. The raw lead payload.
2. Classification output from `lux-lead-classifier`.

## Source files

- `docs/mvps/mvp-b/prompts/user_sequence_commercial.txt`
- `docs/mvps/mvp-b/prompts/user_sequence_residential.txt`
- `docs/mvps/mvp-b/prompts/system_brand_voice.txt`
- `docs/mvps/mvp-b/payload.json`
- `docs/mvps/mvp-b/payload_alt.json`

## Routing

- `commercial_renovation` and `multi_family` use the commercial prompt.
- `custom_home`, `addition_or_remodel`, and `unknown` use the residential prompt.

## Rules

- Never use generic follow-up phrasing.
- Do not use exclamation points.
- Mirror the lead's own language when possible.
- Follow the subject-line constraints from the source prompt.
- Return JSON only with `email_1`, `email_2`, and `email_3`.

## When tuning

- Edit the canonical prompt files under `docs/mvps/mvp-b/prompts/`
- Remind the user to sync the changes to the n8n workflow
- Test both the commercial and residential sample payloads
