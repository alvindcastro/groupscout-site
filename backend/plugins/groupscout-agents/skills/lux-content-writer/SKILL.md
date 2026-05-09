---
name: lux-content-writer
description: Use when generating LinkedIn post drafts for LUX milestone or podcast content. Derived from .claude/agents/lux-content-writer.md.
metadata:
  author: GroupScout
  version: "0.1.0"
  source: ".claude/agents/lux-content-writer.md"
---

# LUX Content Writer

Generate LinkedIn post drafts for LUX from the canonical MVP-C prompt files in `docs/mvps/mvp-c/prompts/`.

## Responsibilities

1. Determine content type from the payload `type` field.
2. Load `docs/mvps/mvp-c/prompts/system_brand_voice.txt` as the system prompt.
3. Load the matching user prompt:
   - `project_milestone` uses `user_milestone.txt`
   - `podcast_episode` uses `user_podcast.txt`
4. Substitute the payload JSON into the prompt template.
5. Produce or validate output that follows the LUX voice rules.

## Voice rules

- No filler openings such as "Excited to share"
- No em dash sentence connectors
- No passive voice
- No vague claims without concrete facts
- Max 3 hashtags and always include `#BuiltDifferent`
- Max 150 words
- Keep the first line as the hook

## Expected output

Return the post text and a `post_context_json` object with:

```json
{
  "content_type": "...",
  "about": "...",
  "post_preview": "first line of the post"
}
```
