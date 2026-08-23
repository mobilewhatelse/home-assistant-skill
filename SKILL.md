# Home Assistant Development Skill

A comprehensive guide for developing Home Assistant automations, dashboards, and integrations using YAML configuration files.

---

## Core Concepts

### Configuration Structure

HA config lives in `/config/` (accessible via Samba at `\\<ha-ip>\config\`). Key files:

```
config/
  automations.yaml          # All automations (list of automation objects)
  configuration.yaml        # Main config, includes other files
  input_boolean.yaml        # Toggle helpers
  input_number.yaml         # Numeric helpers
  input_select.yaml         # Dropdown helpers
  .storage/                 # HA internal state (JSON, do not edit while HA runs)
```

Split helpers into separate files and include them in `configuration.yaml`:

```yaml
# configuration.yaml
input_boolean: !include input_booleans.yaml
input_number: !include input_numbers.yaml
```

### Deployment

Copy files via Samba share:
```powershell
cp automations.yaml "\\192.168.x.x\config\automations.yaml"
```

Reload without restart: **Developer Tools → YAML → Reload Automations**  
New `input_*` helpers require a **full HA restart** to register.

---

## Automations

### Basic Structure

```yaml
- alias: Device - Action Description
  description: When X happens, do Y
  triggers:
  - trigger: <type>
    ...
  conditions:
  - condition: <type>
    ...
  actions:
  - action: <service>
    ...
  mode: single  # single | restart | queued | parallel
```

### Trigger Types

**State trigger** — fires when entity changes state:
```yaml
triggers:
- trigger: state
  entity_id: switch.my_switch
  to: 'on'
  for: "00:00:05"   # optional: must stay in state for duration (debounce)
```

**Numeric state trigger** — fires when value crosses threshold:
```yaml
triggers:
- trigger: numeric_state
  entity_id: sensor.power
  above: 500              # fires when crossing FROM below TO above
  below: 100              # fires when crossing FROM above TO below
```

> **Critical**: `numeric_state` triggers fire only on threshold **crossing**, not when the value is already past the threshold at reload time. Use a "steuerung aktiviert" automation or startup check to handle the already-past-threshold case.

**Time pattern trigger**:
```yaml
triggers:
- trigger: time_pattern
  minutes: "0"    # every hour at :00
  hours: "8"      # at 08:xx
```

**HA startup trigger**:
```yaml
triggers:
- trigger: homeassistant
  event: start
```

### Automation Modes

| Mode | Behavior |
|------|----------|
| `single` | Ignore new triggers while running |
| `restart` | Cancel current run, start fresh on new trigger |
| `queued` | Queue new triggers |
| `parallel` | Run multiple instances simultaneously |

Use `mode: restart` for solar control automations with delay+recheck patterns.  
Use `mode: single` for startup checks and immediate-action automations.

### Delay + Recheck Pattern

The standard pattern for solar device control — avoids false triggers from momentary fluctuations:

```yaml
actions:
- delay:
    minutes: "{{ states('input_number.wartezeit') | int }}"
- condition: numeric_state
  entity_id: sensor.solar_power
  above: input_number.einschalt_watt   # re-check after delay
- action: switch.turn_on
  target:
    entity_id: switch.my_device
mode: restart   # new trigger restarts the delay
```

### Choose (if/else)

```yaml
actions:
- choose:
  - conditions:
    - condition: numeric_state
      entity_id: sensor.power
      above: 500
    sequence:
    - action: select.select_option
      target:
        entity_id: select.device_mode
      data:
        option: High
  - conditions:
    - condition: numeric_state
      entity_id: sensor.power
      above: 200
    sequence:
    - action: select.select_option
      target:
        entity_id: select.device_mode
      data:
        option: Mid
  default:
  - action: select.select_option
    target:
      entity_id: select.device_mode
    data:
      option: Low
```

### Wait Template

Use `wait_template` in startup automations to pause until sensors have valid values:

```yaml
actions:
- delay: "00:01:00"
- wait_template: >
    {{ states('sensor.solar_power') | is_number }}
  timeout: "00:05:00"
  continue_on_timeout: true
- choose:
    ...
```

This prevents `unavailable` sensor values from causing conditions to silently fail at boot.

### Template Conditions

```yaml
- condition: template
  value_template: >
    {{ states('sensor.temperature') | float >
       states('input_number.temp_threshold') | float + 1 }}
```

---

## Input Helpers

### input_number

```yaml
# input_numbers.yaml
my_threshold:
  name: My Threshold
  initial: 250
  min: 0
  max: 2000
  step: 10
  unit_of_measurement: "W"
  icon: mdi:speedometer
```

- `initial:` is only applied on **first entity creation**. After that HA persists the last-set value across restarts.
- Reference in conditions: `above: input_number.my_threshold` (no quotes, no `states()`)
- Reference in templates: `{{ states('input_number.my_threshold') | float }}`

### input_boolean

```yaml
# input_booleans.yaml
my_control:
  name: My Control Active
  initial: true
  icon: mdi:toggle-switch
```

---

## Solar Device Control Pattern

Complete pattern for a device controlled by solar feed-in power:

### Input Helpers

```yaml
# Booleans
device_solar_steuerung:
  name: Device Solar Control
  initial: true
  icon: mdi:solar-power

device_modus_steuerung:
  name: Device Mode Control
  initial: true
  icon: mdi:sun-wireless

# Numbers
device_solar_einschalt_watt:
  name: Device Turn On At (W)
  initial: 250
  min: 0
  max: 2000
  step: 10
  unit_of_measurement: "W"

device_solar_abschalt_watt:
  name: Device Turn Off Under (W)
  initial: 50
  min: 0
  max: 2000
  step: 10
  unit_of_measurement: "W"

device_solar_wartezeit:
  name: Device Solar Wait Time
  initial: 10
  min: 1
  max: 30
  step: 1
  unit_of_measurement: "min"
```

### Automations

```yaml
# Turn on after solar exceeds threshold
- alias: Device - Solar Einschalten
  triggers:
  - trigger: numeric_state
    entity_id: sensor.solar_power
    above: input_number.device_solar_einschalt_watt
  conditions:
  - condition: state
    entity_id: input_boolean.device_solar_steuerung
    state: 'on'
  - condition: state
    entity_id: switch.my_device_socket
    state: 'off'
  actions:
  - delay:
      minutes: "{{ states('input_number.device_solar_wartezeit') | int }}"
  - condition: numeric_state
    entity_id: sensor.solar_power
    above: input_number.device_solar_einschalt_watt
  - action: switch.turn_on
    target:
      entity_id: switch.my_device_socket
  mode: restart

# Turn off after solar drops below threshold
- alias: Device - Solar Ausschalten
  triggers:
  - trigger: numeric_state
    entity_id: sensor.solar_power
    below: input_number.device_solar_abschalt_watt
  conditions:
  - condition: state
    entity_id: input_boolean.device_solar_steuerung
    state: 'on'
  - condition: state
    entity_id: switch.my_device_socket
    state: 'on'
  actions:
  - delay:
      minutes: "{{ states('input_number.device_solar_wartezeit') | int }}"
  - condition: numeric_state
    entity_id: sensor.solar_power
    below: input_number.device_solar_abschalt_watt
  - action: switch.turn_off
    target:
      entity_id: switch.my_device_socket
  mode: restart

# Immediate check when control is toggled on
- alias: Device - Solar-Steuerung aktiviert
  triggers:
  - trigger: state
    entity_id: input_boolean.device_solar_steuerung
    to: 'on'
  conditions: []
  actions:
  - choose:
    - conditions:
      - condition: numeric_state
        entity_id: sensor.solar_power
        above: 100
      sequence:
      - action: switch.turn_on
        target:
          entity_id: switch.my_device_socket
    default:
    - action: switch.turn_off
      target:
        entity_id: switch.my_device_socket
  mode: single
```

### Low/Mid/High Mode Control (e.g. for miners)

```yaml
device_watt_low:
  name: Device Low Mode At (W)
  initial: 0

device_watt_mid:
  name: Device Mid Mode At (W)
  initial: 200

device_watt_high:
  name: Device High Mode At (W)
  initial: 500
```

```yaml
# Start at Low on power-on, upgrade after 2 minutes
- alias: Device - Start mit Low Modus
  triggers:
  - trigger: state
    entity_id: switch.my_device_socket
    to: 'on'
  conditions: []
  actions:
  - delay: "00:00:30"
  - action: select.select_option
    target:
      entity_id: select.device_mode
    data:
      option: Low
  - delay: "00:02:00"
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.device_modus_steuerung
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        above: input_number.device_watt_high
      sequence:
      - action: select.select_option
        target:
          entity_id: select.device_mode
        data:
          option: High
    - conditions:
      - condition: state
        entity_id: input_boolean.device_modus_steuerung
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        above: input_number.device_watt_mid
      sequence:
      - action: select.select_option
        target:
          entity_id: select.device_mode
        data:
          option: Mid
  mode: single   # IMPORTANT: single, not restart — avoids p1_status flicker loops

# Switch to High when solar exceeds High threshold
- alias: Device - Modus High
  triggers:
  - trigger: numeric_state
    entity_id: sensor.solar_power
    above: input_number.device_watt_high
  conditions:
  - condition: state
    entity_id: input_boolean.device_modus_steuerung
    state: 'on'
  actions:
  - delay:
      minutes: "{{ states('input_number.device_solar_wartezeit') | int }}"
  - condition: numeric_state
    entity_id: sensor.solar_power
    above: input_number.device_watt_high
  - action: select.select_option
    target:
      entity_id: select.device_mode
    data:
      option: High
  mode: restart
```

---

## Startup Check Pattern

Runs at HA start to set all devices to their correct state:

```yaml
- alias: System - Startup Check
  triggers:
  - trigger: homeassistant
    event: start
  conditions: []
  actions:
  - delay: "00:01:00"
  - wait_template: >
      {{ states('sensor.solar_power') | is_number }}
    timeout: "00:05:00"
    continue_on_timeout: true

  # Device 1 solar
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.device_solar_steuerung
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        above: input_number.device_solar_einschalt_watt
      sequence:
      - action: switch.turn_on
        target:
          entity_id: switch.my_device_socket
    - conditions:
      - condition: state
        entity_id: input_boolean.device_solar_steuerung
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        below: input_number.device_solar_abschalt_watt
      sequence:
      - action: switch.turn_off
        target:
          entity_id: switch.my_device_socket

  # Device 1 mode
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.device_modus_steuerung
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        above: input_number.device_watt_high
      sequence:
      - action: select.select_option
        target:
          entity_id: select.device_mode
        data:
          option: High
    - conditions:
      - condition: state
        entity_id: input_boolean.device_modus_steuerung
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        above: input_number.device_watt_mid
      sequence:
      - action: select.select_option
        target:
          entity_id: select.device_mode
        data:
          option: Mid
    - conditions:
      - condition: state
        entity_id: input_boolean.device_modus_steuerung
        state: 'on'
      sequence:
      - action: select.select_option
        target:
          entity_id: select.device_mode
        data:
          option: Low
  mode: single
```

---

## Periodic Check Pattern

Hourly correction to catch missed triggers (e.g. after automation reload while solar was already past threshold):

```yaml
- alias: System - Periodischer Check
  triggers:
  - trigger: time_pattern
    minutes: "0"
  conditions: []
  actions:
  # Turn off if solar too low (device is on but shouldn't be)
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.device_solar_steuerung
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        below: input_number.device_solar_abschalt_watt
      - condition: state
        entity_id: switch.my_device_socket
        state: 'on'
      sequence:
      - action: switch.turn_off
        target:
          entity_id: switch.my_device_socket
  # Correct mode
  - choose:
    - conditions:
      - condition: state
        entity_id: input_boolean.device_modus_steuerung
        state: 'on'
      - condition: state
        entity_id: switch.my_device_socket
        state: 'on'
      - condition: numeric_state
        entity_id: sensor.solar_power
        above: input_number.device_watt_high
      sequence:
      - action: select.select_option
        target:
          entity_id: select.device_mode
        data:
          option: High
    ...
  mode: single
```

---

## Dashboard YAML (Lovelace)

### Sections Layout

```yaml
views:
  - type: sections
    title: My View
    icon: mdi:home
    max_columns: 4
    show_icon_and_title: true
    cards: []
    sections:
      - type: grid
        cards:
          - type: vertical-stack
            cards:
              - type: glance
                title: Status Overview
                entities:
                  - entity: sensor.my_sensor
                    name: Value
                    icon: mdi:gauge
              - type: entities
                title: Controls
                show_header_toggle: false
                entities:
                  - entity: input_boolean.my_control
                    name: Auto Mode
                    icon: mdi:auto-fix
                  - entity: input_number.my_threshold
                    name: Threshold
              - type: markdown
                content: >
                  ℹ️ Info text here
```

### Dashboard Storage

HA stores dashboard config in `.storage/lovelace.<dashboard_id>`. The dashboard must be registered in `.storage/lovelace_dashboards`:

```json
{
  "id": "my_dashboard",
  "show_in_sidebar": true,
  "icon": "mdi:home",
  "title": "My Dashboard",
  "require_admin": false,
  "mode": "storage",
  "url_path": "my-dashboard"
}
```

The `.storage` key matches the `id`: `lovelace.my_dashboard`.

---

## .storage Files

HA persists integration config in `.storage/core.config_entries`. Each integration has a `data` block with connection parameters (host, port, credentials).

**Do not try to change integration config by editing this file.** HA holds config entries in memory and writes them back to disk on its own schedule. An edit made while HA runs gets silently reverted — often within minutes, with no restart involved. An integration stuck in `setup_retry` is retried continuously and each retry can trigger a store write, so a failing integration is exactly the case where file edits are least likely to survive.

Editing while HA is stopped does work, but only if you can guarantee HA stays down for the edit and you have a way to start it again. On a remote/headless box that is usually not worth it.

Use the REST API instead (see below). It changes HA's in-memory state, which is the authoritative copy, and HA persists it itself.

Lovelace dashboards are the exception worth knowing: `.storage/lovelace.<id>` files can be created or edited directly, because HA reads them on demand rather than holding them in a write-back cache. After editing, reload via **Developer Tools → YAML → Reload Lovelace dashboards** or restart. Editing the dashboard in the UI afterwards will overwrite your file.

---

## REST API

Far more reliable than poking at files, and it removes the need to ask the user to click through the UI. Create a token under **Profile → Long-Lived Access Tokens**.

```bash
TOKEN="eyJhbGci..."
HA="http://homeassistant.local:8123"
AUTH=(-H "Authorization: Bearer $TOKEN")
```

### Read state — verify instead of assuming

```bash
curl -s "${AUTH[@]}" "$HA/api/states/sensor.my_sensor"
curl -s "${AUTH[@]}" "$HA/api/states"          # everything
```

Use this to confirm an entity actually exists and holds a sane value before writing automations against it. Typos in entity IDs are the single most common cause of automations that "run but do nothing".

### Call services

```bash
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" \
  -d '{"entity_id":"switch.my_switch"}' "$HA/api/services/switch/turn_on"
```

### Inspect integrations

```bash
curl -s "${AUTH[@]}" "$HA/api/config/config_entries/entry?domain=my_integration"
```

Returns one object per entry. The fields that decide what you can change:

| Field | Meaning |
|---|---|
| `entry_id` | Handle for options/reconfigure flows |
| `state` | `loaded`, `setup_retry`, `setup_error`, … |
| `reason` | Why setup failed |
| `supports_options` | An options flow exists |
| `supports_reconfigure` | A reconfigure flow exists (can change connection settings) |

### Drive a config or options flow

Options flows are multi-step. Start one with the `entry_id` as handler, then POST each step's answers to the returned `flow_id`:

```bash
# Start — returns flow_id and the first step's schema
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" \
  -d '{"handler":"01ABCDEF..."}' "$HA/api/config/config_entries/options/flow"

# Answer a step
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" \
  -d '{"action":"host"}' "$HA/api/config/config_entries/options/flow/$FLOW_ID"

# Abandon an unfinished flow
curl -s -X DELETE "${AUTH[@]}" \
  "$HA/api/config/config_entries/options/flow/$FLOW_ID"
```

The response `type` tells you where you are: `form` (another step, schema included), `create_entry` (done), or `abort` (done, with a `reason`). The returned `data_schema` shows the exact field names to send — read it rather than guessing.

### Restart and wait

```bash
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" \
  -d '{}' "$HA/api/services/homeassistant/restart"
```

Then poll `$HA/api/config` until `state` is `RUNNING`. The sequence is connection-refused → `NOT_RUNNING` → `STARTING` → `RUNNING`, typically 60–120 s. Do not test anything before `RUNNING` — entities are still materializing and everything looks `unavailable`.

---

## Patching a Custom Component

Sometimes an integration simply cannot do what you need. A concrete and common case: a device gets a new DHCP IP, but the integration offers no way to change the host — `supports_reconfigure` is `false` and the options flow only covers unrelated settings.

Check the alternatives first, because patching is a maintenance burden:

- Give the device a static IP or a DHCP reservation on the router. This is the real fix — do it if the router allows it.
- Use a hostname instead of an IP in the integration config, if the device registers one via mDNS/DNS.
- Delete and re-add the integration. Check the `config_flow.py` first: if `unique_id` is derived from the host (`async_set_unique_id(host.replace(".", "_"))`), re-adding under a new IP produces a **new** unique_id, therefore new entity IDs, breaking every automation, template and dashboard that referenced the old ones. That usually rules it out.

If you do patch, adding a step to the options flow is a small, contained change:

```python
async def async_step_host(self, user_input=None) -> FlowResult:
    """Change the device host/IP (e.g. after a DHCP change)."""
    errors = {}
    current_host = self._config_entry.data.get(CONF_HOST, "")

    if user_input is not None:
        new_host = user_input.get(CONF_HOST, "").strip()
        if not new_host:
            errors[CONF_HOST] = "empty_host"
        elif new_host == current_host:
            return self.async_abort(reason="host_unchanged")
        else:
            new_data = dict(self._config_entry.data)
            new_data[CONF_HOST] = new_host
            # Writes entry.data AND persists it — unique_id untouched,
            # so all entity IDs survive.
            self.hass.config_entries.async_update_entry(
                self._config_entry, data=new_data, title=new_host,
            )
            self.hass.config_entries.async_schedule_reload(
                self._config_entry.entry_id
            )
            return self.async_abort(
                reason="host_updated",
                description_placeholders={"host": new_host},
            )

    return self.async_show_form(
        step_id="host",
        data_schema=vol.Schema({
            vol.Required(CONF_HOST, default=current_host): str
        }),
        errors=errors,
    )
```

Key points:

- **`async_update_entry(data=...)` is the right tool.** It updates the in-memory entry and persists it. This is what file editing was trying and failing to do.
- **Leave `unique_id` alone.** Entity IDs are derived from it. Changing it orphans every entity.
- **`data` vs `options`.** Connection settings live in `entry.data`; user preferences live in `entry.options`. `async_create_entry()` in an options flow writes `options`, so a host change needs `async_update_entry(data=...)` instead.
- Wire the step into `async_step_init` and add it to the action selector.
- Add `translations/*.json` entries for the new step, the selector option, and any `abort`/`error` reasons. A missing translation key surfaces as a raw string in the UI, not an error, so it is easy to miss.

Then:

- **Python changes in a custom component need a full HA restart.** Reloading the config entry re-runs setup with the already-imported module; it does not re-import your edited file.
- **HACS updates overwrite custom component files.** Keep patched files in your own repo alongside a note on how to redeploy, or the patch will vanish at the next update with no warning.

---

## Common Pitfalls

### Duplicate YAML keys
YAML last-key-wins — duplicate keys silently override. Always check for accidental duplicates after edits.

```yaml
# BUG: mode: single overrides mode: restart
- alias: My Automation
  ...
  mode: restart
  mode: single    # this wins — restart never applies
```

### numeric_state trigger already past threshold
After automations reload, `numeric_state` triggers won't fire if the value is already past the threshold. Always pair with a "steuerung aktiviert" automation and a startup check.

### p1_status flicker causes constant Low mode
If a device has a status sensor that flickers (e.g. connection status briefly drops), using it as a trigger with `mode: restart` causes the automation to restart constantly and never finish. Fix: use `mode: single` and/or add `for: "00:00:05"` to debounce.

### New input_* helpers need HA restart
Reloading YAML only works for automations and scripts. New `input_boolean`, `input_number`, `input_select` helpers require a full restart to register their entities.

### Conditions silently fail on unavailable sensors
At HA boot, sensors may be `unavailable` for 1-3 minutes. `numeric_state` conditions on unavailable sensors return `false` silently. Use `wait_template` in startup automations.

### choose: requires correct indentation
The `choose:` action is an action, not a key — it must be at action level:
```yaml
actions:
- choose:          # correct: dash before choose
  - conditions:
    ...
```

---

## AI Integration

Use `ai_task.generate_data` for intelligent notifications:

```yaml
- action: ai_task.generate_data
  data:
    task_name: claude_ai_task
    instructions: >
      Analyze the following sensor values and report any anomalies.
      Current values:
      - Power: {{ states('sensor.power') }} W
      - Temperature: {{ states('sensor.temperature') }} °C
      Provide a brief diagnosis and recommended action.
  response_variable: ai_response
- action: persistent_notification.create
  data:
    title: "🚨 Alert {{ now().strftime('%d.%m.%Y %H:%M') }}"
    message: "{{ ai_response.data }}"
- action: notify.mobile_app_iphone
  data:
    title: "🚨 Alert"
    message: "{{ ai_response.data }}"
```

---

## Useful Template Filters

```yaml
# String to float
{{ states('sensor.power') | float }}
{{ states('sensor.power') | float(0) }}  # default 0 if unavailable

# String to int
{{ states('input_number.minutes') | int }}

# Check if numeric (not unavailable/unknown)
{{ states('sensor.power') | is_number }}

# Current time
{{ now().strftime('%d.%m.%Y %H:%M') }}

# Compare with input_number (in template condition)
{{ states('sensor.temp') | float > states('input_number.threshold') | float + 1 }}
```

---

## HACS Integrations

HACS (Home Assistant Community Store) adds custom integrations. Config is persisted in `.storage/core.config_entries` under the integration domain.

- **Port 4028** (CGMiner protocol): Used by mining hardware integrations for **mode control** via `select` entities
- **REST sensors** (HTTP): Independent of HACS, used for monitoring data — survives integration reinstall
- Integration config persists in `.storage` even after UI deletion — full HA restart restores it

---

## Entity Naming Conventions

HA auto-generates entity IDs from device/entity names:
- Spaces → underscores
- Umlauts: ü→u, ä→a, ö→o (or kept as-is depending on integration)
- Prefix from device name, suffix from entity type

Always verify entity IDs in **Developer Tools → States** rather than assuming from the name.

For Tuya combo devices (socket + temperature sensor), the temperature entity often uses a mix of languages:
- Some devices: `sensor.device_name_temperature` (English)
- Others: `sensor.device_name_temperatur` (German)

Click on the entity in **Settings → Devices → [Device] → Temperatur** to see the exact entity ID.

---

## Multi-File Include with Packages

When a single YAML file defines multiple top-level keys (`input_select`, `input_number`, `automation`, `template`, etc.), you **cannot** use `!include` directly at the top level — it needs a key. Use `packages` instead:

```yaml
# configuration.yaml
homeassistant:
  packages:
    my_feature: !include my_feature.yaml
```

The included file can contain any combination of top-level keys:

```yaml
# my_feature.yaml
input_select:
  my_mode: ...

input_number:
  my_threshold: ...

automation:
  - id: my_automation
    ...

template:
  - sensor:
      ...
```

**Never** write `!include file.yaml` as a standalone line — YAML requires a key before every value.

---

## Multi-Room Thermostat Control Pattern

Complete pattern for controlling multiple heaters (Tuya smart plugs with built-in temperature sensors) via 4 operating modes.

### Modes

| Mode | Behavior |
|------|----------|
| `Aus` | All heaters off immediately |
| `Winter` | Frost protection — hold constant temp (e.g. 8°C), ±0.3°C hysteresis |
| `Urlaub` | Day/Night schedule — higher temp by day, lower at night, configurable hours |
| `Test` | Manual control of each heater individually |

### Input Helpers

```yaml
input_select:
  heizung_modus:
    name: "Heizungsmodus"
    options: ["Aus", "Winter", "Urlaub", "Test"]
    icon: mdi:fire

input_number:
  winter_temperatur:
    name: "Winter Zieltemperatur"
    min: 5
    max: 25
    step: 0.5
    unit_of_measurement: "°C"
    initial: 8

  urlaub_temp_tag:
    name: "Urlaub Tagestemperatur"
    min: 15
    max: 25
    step: 0.5
    unit_of_measurement: "°C"
    initial: 22

  urlaub_temp_nacht:
    name: "Urlaub Nachttemperatur"
    min: 10
    max: 20
    step: 0.5
    unit_of_measurement: "°C"
    initial: 18

  urlaub_start_tag:
    name: "Urlaub Tagesstart (Stunde)"
    min: 0
    max: 23
    step: 1
    unit_of_measurement: "h"
    initial: 6

  urlaub_start_nacht:
    name: "Urlaub Nachtstart (Stunde)"
    min: 0
    max: 23
    step: 1
    unit_of_measurement: "h"
    initial: 23

input_boolean:
  test_room1:
    name: "TEST: Room 1"
    icon: mdi:toggle-switch
```

### Thermostat Automation with Hysteresis

Use `condition: template` instead of `numeric_state` for dynamic thresholds in action `if` blocks — `above`/`below` do NOT accept templates there:

```yaml
- id: heizung_winter_room1
  alias: "Heizung - WINTER Room 1"
  trigger:
    - platform: state
      entity_id: sensor.room1_temperature
    - platform: time_pattern
      minutes: "/10"
  condition:
    - condition: state
      entity_id: input_select.heizung_modus
      state: "Winter"
  action:
    - if:
        - condition: template
          value_template: >
            {{ states('sensor.room1_temperature') | float(99) <
               (states('input_number.winter_temperatur') | float(8)) - 0.3 }}
      then:
        - service: homeassistant.turn_on
          target:
            entity_id: switch.room1_heater
      else:
        - if:
            - condition: template
              value_template: >
                {{ states('sensor.room1_temperature') | float(0) >
                   (states('input_number.winter_temperatur') | float(8)) + 0.3 }}
          then:
            - service: homeassistant.turn_off
              target:
                entity_id: switch.room1_heater
```

**Hysteresis values:**
- Winter: ±0.3°C → 0.6°C total band (prevents rapid switching at low temps)
- Holiday: ±0.5°C → 1.0°C total band

### Day/Night Time Condition (handles midnight wraparound)

```yaml
# Daytime condition (e.g. 06:00–23:00)
- condition: template
  value_template: >
    {% set start = states('input_number.urlaub_start_tag') | int(6) %}
    {% set end = states('input_number.urlaub_start_nacht') | int(23) %}
    {{ now().hour >= start and now().hour < end }}

# Nighttime condition (handles wraparound: e.g. 23:00–06:00)
- condition: template
  value_template: >
    {% set start = states('input_number.urlaub_start_nacht') | int(23) %}
    {% set end = states('input_number.urlaub_start_tag') | int(6) %}
    {% set hour = now().hour %}
    {% if start > end %}
      {{ hour >= start or hour < end }}
    {% else %}
      {{ hour >= start and hour < end }}
    {% endif %}
```

### Template Sensors for Dashboard

```yaml
template:
  - sensor:
      - name: "Heizung Status"
        unique_id: heizung_status
        state: "{{ states('input_select.heizung_modus') }}"

      - name: "Aktive Heizungen"
        unique_id: aktive_heizungen
        state: >
          {% set on_count = [
            states('switch.room1_heater'),
            states('switch.room2_heater')
          ] | select('equalto', 'on') | list | length %}
          {{ on_count }}

      - name: "Durchschnittstemperatur"
        unique_id: durchschnitts_temperatur
        unit_of_measurement: "°C"
        device_class: temperature
        state: >
          {% set temps = [
            states('sensor.room1_temperature') | float(0),
            states('sensor.room2_temperature') | float(0)
          ] | select('>', 0) | list %}
          {{ (temps | sum / temps | length) | round(1) if temps else 'N/A' }}
```

---

## Dashboard Deployment via Raw Editor

Lovelace dashboards are NOT included in `configuration.yaml`. Upload them via the UI:

1. **Settings → Dashboards → + Add Dashboard** → choose type (Masonry)
2. Navigate to the new dashboard in the sidebar
3. Top right **⋮ → Edit Dashboard → ⋮ → Raw configuration editor**
4. Select all (`Ctrl+A`) → delete → paste YAML → Save

**Important:** Always select-all and delete first. Pasting without clearing causes `duplicated mapping key` errors (two `title:` keys).

### Tile Cards (recommended over button/entities for switches)

`type: tile` shows entity name, current state/value, and toggles for switches — more compact than `type: button`:

```yaml
- type: tile
  entity: switch.room1_heater
  name: "Room 1"
  icon: mdi:radiator

- type: tile
  entity: sensor.room1_temperature
  name: "Room 1 Temp"
```

### Valid Lovelace View Types

```yaml
views:
  - path: main
    title: "My View"
    # type: masonry   ← default, omit or use explicitly
    # type: panel
    # type: sidebar
    # type: sections
    cards: [...]
```

`type: vertical` does NOT exist — omit `type` to get the default masonry layout.

---

## New Common Pitfalls

### `description:` not valid for input helpers
`input_boolean`, `input_select`, and `input_number` do not accept a `description:` field — HA will warn and ignore the entity. Remove it.

### `initial_value:` vs `initial:` for input_number
The correct key is `initial:`, not `initial_value:`.

```yaml
# Wrong
my_number:
  initial_value: 8

# Correct
my_number:
  initial: 8
```

### Templates not allowed in `above:`/`below:` inside action `if` blocks
`numeric_state` conditions inside action `if`/`then`/`else` blocks do not support Jinja2 templates for `above` and `below`. Use `condition: template` instead:

```yaml
# Wrong — causes "expected float" error
- if:
    - condition: numeric_state
      entity_id: sensor.temp
      below: "{{ states('input_number.target') | float - 0.3 }}"

# Correct
- if:
    - condition: template
      value_template: >
        {{ states('sensor.temp') | float(99) <
           states('input_number.target') | float - 0.3 }}
```

### `!include` as standalone line
`!include` must be the value of a key — never a standalone line:

```yaml
# Wrong — causes "multiline key may not be an implicit key" error
!include my_file.yaml

# Correct
my_section: !include my_file.yaml

# For multi-key files, use packages:
homeassistant:
  packages:
    my_feature: !include my_feature.yaml
```

### "Entities unavailable" is usually an integration problem, not an entity problem

When a device's entities all read `unavailable`, do not start by inspecting entities or dashboards. Check the integration entry first:

```bash
curl -s "${AUTH[@]}" "$HA/api/config/config_entries/entry?domain=my_integration" \
  | python -c "import json,sys
for e in json.load(sys.stdin):
    print(e['title'], e['state'], e['reason'])"
```

A `state` of `setup_retry` with a `reason` points straight at the cause — wrong IP, wrong credentials, device offline. A cryptic `reason` (e.g. `'parsed'`, a raw `KeyError` from the integration) still tells you setup is failing rather than the entities being misconfigured.

### Keep a feature's helpers and automations in one package file

When adding a self-contained feature, resist appending its helpers to whatever existing `input_boolean:`/`input_number:` include happens to be there. Give it a package:

```yaml
# configuration.yaml
homeassistant:
  packages:
    poolpump: !include poolpump.yaml
```

```yaml
# poolpump.yaml — helpers and automations together
input_boolean:
  poolpump_solar_control:
    name: Pool Pump Solar Control
    initial: true

automation:
  - alias: Pool Pump - Solar control on at startup
    triggers:
      - trigger: homeassistant
        event: start
    actions:
      - action: input_boolean.turn_on
        target:
          entity_id: input_boolean.poolpump_solar_control
```

The feature becomes one file to read, move, or delete, and unrelated features stop accumulating each other's helpers. Note that `automation:` inside a package coexists with `automation: !include automations.yaml` — packages merge rather than collide, so UI-created automations keep working.

### Retrofitting a manual override onto existing automations

To make existing automations skippable without rewriting them, add a single state condition at the top of each:

```yaml
conditions:
  - condition: state
    entity_id: input_boolean.my_feature_auto_control
    state: 'on'
  # ... existing conditions unchanged
```

Two things this needs to be complete:

- **Set the boolean explicitly at startup** if one state is meant to be the default after a restart. `initial:` covers helper creation, but being explicit in the startup automation makes the intent visible and survives someone later removing `initial:`.
- **Add a "control re-enabled" automation.** Turning the boolean back on does not re-fire the original `numeric_state` triggers, so the device keeps whatever state manual mode left it in until the next threshold crossing. Trigger on the boolean going `on` and apply the correct state immediately — the same reason startup checks are needed.
