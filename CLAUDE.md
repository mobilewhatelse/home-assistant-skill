# Home Assistant Skill Repo

This repo contains one Claude Code skill (`SKILL.md`) plus a README. Language of the content: English.

## Content policy (hard rules)
This repository is public. Only general, person-free insights belong here: no IP addresses, serial numbers, names, tokens,
URLs of a real installation, or entity IDs that identify a person. Use placeholders (`<ha-ip>`, `sensor.example_power`).

## SKILL.md frontmatter
Use **only the frontmatter fields that both Claude Code and GitHub Copilot document**:

```yaml
---
name: home-assistant
description: <one or two sentences - what it covers and when to invoke it>
---
```

- `name` and `description` (10-1024 characters, plain single-line text without `: ` or ` #`) are required; `license` is optional.
- **Do not add other fields.** `argument-hint`, `user-invocable`, `disable-model-invocation`, and `allowed-tools` are not understood by every harness;
  GitHub Copilot does not support them in a skill. Tools and subagents belong in a custom agent or prompt file, not in a skill.

## Layout note
The skill currently lives at the repo root (`SKILL.md`), which suits a manual copy into a Claude Code skills folder. GitHub Copilot discovers
project skills under `.github/skills/<name>/SKILL.md`; moving the file there is the step to make it discoverable by Copilot without copying.
