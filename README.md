# Skills

Public opencode skills collection.

## Skills

- `review-babysit` — Drive a GitHub PR to zero actionable Copilot feedback: request review, address every comment, re-request, repeat. See `review-babysit/SKILL.md`.

## Usage

opencode auto-loads `**/SKILL.md`. To use from another repo, add to `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["../skills"]
  }
}
```

Or copy the skill folder into `.opencode/skills/`.
