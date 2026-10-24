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
- iPhone home-screen widgets with Scriptable: one template request, token in
  the Keychain, and the pitfalls of getting a script onto the phone

**Automations with more than one reason to run**
- Two independent enable switches without one silently disabling the other
- Modelling reasons explicitly, switching off only when none holds
- Threshold triggers are edge-triggered — and only as fast as their source
  sensor (how to measure that, and what it costs you)

**Battery/inverter integration**
- Emulating a smart meter for a battery that only discovers meters itself
  (protocol choice, mandatory mDNS, why identity values must look like a
  real device, the "found but offline" trap, and restart-safe re-registration)
- Testing a device's internal rules with a real load instead of a faked
  reading — and why a faked one cannot answer some questions at all
- Settings that snap back silently, settings whose names don't match what
  they do, and recovering per-unit values from an aggregate
- A follow-meter control loop that oscillates: checking the update rate
  against the device's polling rate before touching the fed value, and why
  "correcting" the value to cancel the device's own contribution is a
  tempting but wrong fix that hides the oscillation while silently
  overproducing

**When the network changes**
- Recovering after a router renumbers devices: finding all stale addresses,
  how reconfigure / options flows / delete-and-re-add differ, values that revert
  on restart, scanning for moved devices, per-address Windows credentials
- Verifying a data-source swap in both operating regimes (generating and not)

**Diagnosis and housekeeping**
- Telling a device-specific daily dropout from a network one (compare other devices,
  host journal, router maintenance cycles)
- Spotting a redundant integration before patching it, and cleaning up snapshot
  copies and debug scripts safely
- Helpers created over the API: why `initial` resets them on every restart, and
  deciding on a short average (statistics helper, step vs. linear) instead of a
  jumpy instant reading

**Tuya devices that regulate themselves**
- Letting a thermostat socket run its own setpoint and weekly program so a
  network outage does not stop the heating; HA only sets and verifies the mode
- Reaching data points the core Tuya integration does not expose (Tuya
  Developer Platform, API Explorer, signed cloud calls from pyscript)
- Pitfalls: acknowledged-but-ignored writes, the switch that is not the relay,
  odd factory defaults, the raw weekly-program format, not resending after a restart
  (but rebuilding the display read-only)

**Claude Platform and grid tariffs**
- A daily house report written by Claude from metering data: computed in code,
  phrased by the model, read-only tools, structured output, caching, cost per run,
  off after restart
- Power-based grid tariffs: measuring quarter-hour peaks, telling counter jumps
  from real peaks, and a guard that throttles the wallbox and other loads only as
  far as needed (phase-switch pauses, hysteresis, offline simulation)
- pyscript reloads that leave a control loop running twice, and a single-instance guard
- Turning such a guard into a custom integration: engine without HA imports and an offline house model,
  devices by role (wallbox, switch, script), observe mode and lock entity, deciding on averages so cycling
  cookers do not cause flapping, and the pitfalls on the way (restart for code changes, a selector that
  fails with a bare HTTP 400, a duplicate `script:` key that silently drops UI scripts)

**Custom energy dashboard and widgets**
- An animated energy-flow card (button-card v7: style format, automatic entity
  tracking, redraw timer, separate phone layout) and period sums that follow the
  Energy date picker, plus a view theme for the standard energy cards
- Scriptable widgets that draw a static flow picture; day view vs. momentary view
  of "where does the power come from"; offline render loop; the one-line-script trap
- A wallbox that reports "Complete" with the car not full, and a push when sun is
  available but nothing charges

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

## Contributing

Contributions are welcome. [CONTRIBUTING.md](CONTRIBUTING.md) explains how to add or change a skill (portable frontmatter, structure, content policy).
