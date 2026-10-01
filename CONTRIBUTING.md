# Contributing to Home Assistant Skill

Thanks for helping. This repository holds skills for **Claude Code** and **GitHub Copilot**. The same files must load unchanged in both, so a few rules are strict. Please read this page before adding or changing a skill.

## Changing the skill

The skill is the single file `SKILL.md` at the repository root. Add new knowledge as a new section in the matching topic area, keep the existing structure, and keep the content general (see the content policy below).

## Frontmatter (only these fields)

```yaml
---
name: home-assistant-<topic>
description: <one or two sentences - what it covers and when to invoke it>
license: MIT
---
```

`name` and `description` are required (description: 10-1024 characters, one line, no `: ` and no ` #`); `license` is optional. Nothing else.

## Layout note

GitHub Copilot discovers project skills under `.github/skills/<name>/SKILL.md`. The skill currently lives at the repository root, which suits a manual copy into a Claude Code skills folder. Moving the file to the Copilot location is the step that makes it discoverable there without copying.

## Why only three frontmatter fields

GitHub Copilot documents `name`, `description`, and `license` (and `allowed-tools` as plain text) for a skill; fields such as `argument-hint`, `user-invocable`, or `disable-model-invocation` are not supported there, and a user reported that skills carrying them did not work in Copilot. `allowed-tools` would also pre-approve shell access, which documentation-only skills do not need. Keep the portable set.

## Tools and subagents do not belong in a skill

A skill is on-demand knowledge. It cannot restrict tools or start subagents. If you need a fixed tool set or a helper agent, put that in a custom agent (`.github/agents/*.agent.md`, fields `tools`, `agents`) or a prompt file (`.github/prompts/*.prompt.md`) that uses the skill.

## Content policy

These repositories are public and reusable. Do not include:

- user names, e-mail addresses, passwords, tokens, or other credentials,
- organization, customer, project, or environment names, and real identifiers (IDs, GUIDs, domains, IP addresses),
- screenshots or exports of a real environment.

Use placeholders (for example `<workspace-id>` or `Firstname Lastname`) and write what you learned as general mechanics, not as a story about one project.

## Commits

Please use an anonymous or noreply e-mail address for commits to these public repositories.

## Pull request checklist

- [ ] Frontmatter has only `name`, `description`, `license`
- [ ] No names, addresses, credentials, environment identifiers, or links to a real environment
