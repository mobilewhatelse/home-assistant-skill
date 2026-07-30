# Home Assistant Claude Code Skill

A Claude Code skill for developing Home Assistant automations, dashboards, and integrations.

## What's in this skill

- Solar-based device control patterns (turn on/off based on feed-in power)
- Low/Mid/High mode switching for adjustable devices
- Startup check and periodic check patterns
- `wait_template` for robust boot behavior
- Dashboard YAML (Lovelace sections layout)
- `.storage` file management
- AI notification integration (`ai_task.generate_data`)
- Common pitfalls and their fixes

## Usage

Add to your Claude Code project via `.claude/agents/` or reference directly as a skill.

## Install

```bash
gh repo clone mobilewhatelse/home-assistant-skill
```

Then copy `SKILL.md` to your project's `.claude/` directory or reference it in your Claude Code skill configuration.
