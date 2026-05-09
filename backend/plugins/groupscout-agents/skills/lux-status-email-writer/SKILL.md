---
name: lux-status-email-writer
description: Use when generating client status email drafts for LUX from JobTread project payloads. Derived from .claude/agents/lux-status-email-writer.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/lux-status-email-writer.md"
---

# LUX Status Email Writer

Generate client-facing project status emails for LUX from the canonical MVP-A prompts and sample payloads.

## Source files

- `docs/mvps/mvp-a/payload.json`
- `docs/mvps/mvp-a/payload_alt.json`
- `docs/mvps/mvp-a/prompts/system_brand_voice.txt`
- `docs/mvps/mvp-a/prompts/user_status_email.txt`
- `docs/mvps/mvp-a/prompts/user_slack_notify.txt`

## Core rules

- Budget deltas inside plus or minus 5 percent are not mentioned
- Mention only the current in-progress milestone and the next upcoming one when relevant
- Surface client-facing asks as `One thing we need from you:`
- Suppress internal-only LUX items
- Avoid jargon, passive voice, and vague positivity
- Keep the body under 200 words

## Output

Return exactly:

```json
{
  "subject": "Hartwell Residence - Framing Update - May 7",
  "body": "Hi Sarah,\n\n..."
}
```

No preamble or markdown fences.
