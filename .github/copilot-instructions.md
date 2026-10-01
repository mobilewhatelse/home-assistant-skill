# Instructions for AI assistants working in this repository

This repository contains skills for Claude Code and GitHub Copilot. When you add or change a skill, follow `CONTRIBUTING.md`. In short:

- Skill frontmatter contains **only** `name`, `description`, and optionally `license`. Never add `allowed-tools`, `argument-hint`, `user-invocable`, or other fields.
- `description`: 10-1024 characters, one line, no `: ` and no ` #`; say what the skill covers and when to use it.
- Keep content general: no names, e-mail addresses, credentials, organization or environment names, identifiers, or links to a real environment.
- Tools and subagents belong in custom agents or prompt files, not in skills.
