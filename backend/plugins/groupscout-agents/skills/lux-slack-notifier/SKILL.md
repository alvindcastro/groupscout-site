---
name: lux-slack-notifier
description: Use after lux-content-writer to build the internal Slack review payload for MVP-C. Derived from .claude/agents/lux-slack-notifier.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/lux-slack-notifier.md"
---

# LUX Slack Notifier

Build the internal Slack notification payload for the LUX LinkedIn post pipeline after `lux-content-writer` produces a draft.

## Inputs

The upstream `post_context_json`:

```json
{
  "content_type": "project_milestone | podcast_episode",
  "about": "human-readable description",
  "post_preview": "first line of the generated post"
}
```

## Source file

- `docs/mvps/mvp-c/prompts/user_slack_notify.txt`

## Output

Produce a Slack Block Kit payload with:

- A short internal review message
- The full post text
- Action buttons for post, schedule, and edit

## Rules

- The review message is 2 to 3 sentences
- Include the content type, what it is about, and a review note
- Keep the tone internal and casual
- End with a question asking who will review
