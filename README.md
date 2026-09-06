# Home Assistant Claude Code Skill

A Claude Code skill for developing Home Assistant automations, dashboards, and integrations.

## What's in this skill

**Automations and patterns**
- Solar-based device control (turn on/off based on feed-in power)
- Low/Mid/High mode switching for adjustable devices
- Startup check and periodic check patterns, `wait_template` for robust boot
- Multi-room thermostat control with hysteresis
- Retrofitting a manual-override switch onto existing automations
- Packages for self-contained features

**Energy metering**
- Battery behind the meter: why computed house consumption breaks once a
  battery is added, and how to correct it — including preparing the correction
  before the hardware arrives
- Energy dashboard configuration over WebSocket, incl. batteries and
  per-device breakdown
- Creating helpers (Riemann sum, template, utility meter) over the API
- Comparing your figures against the manufacturer's app without chasing
  phantom errors

**Driving HA from outside**
- REST API: reading state, driving config/options flows, restarting and waiting
- WebSocket API: dashboards without file access, Supervisor and add-on
  management, add-on logs, `system_log/list`
- Working against a Nabu Casa remote URL when off the LAN

**Integrations and dashboards**
- Patching a HACS custom component (e.g. adding a host field to an options
  flow) without breaking entity IDs — and re-applying it after an update
  (including: verifying the fix actually reached the live instance, not just
  your own repo)
- Renaming entities in the registry to fix a broken dashboard after a device
  is re-created, instead of rewiring every card and automation
- Consolidating several dashboards into one, and reordering the sidebar
  (the real per-user data key, and its quirks)
- Dashboard YAML (Lovelace sections layout)
- `.storage` files — and why editing them is the wrong tool for integration config
- AI notification integration (`ai_task.generate_data`)

**Diagnostics**
- A `loaded` config entry with dead entities: reading errnos, reloading entries
- Automations without an explicit `id:` spawning duplicate registry entries
- Common pitfalls and their fixes

## Usage

Add to your Claude Code project via `.claude/agents/` or reference directly as a skill.

## Install

```bash
gh repo clone mobilewhatelse/home-assistant-skill
```

Then copy `SKILL.md` to your project's `.claude/` directory or reference it in your Claude Code skill configuration.
