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

HA persists integration config in `.storage/core.config_entries`. Each integration has a `data` block with connection parameters. To update a connection (e.g. IP change):

```bash
sed -i 's/"host":"192.168.1.100"/"host":"192.168.1.101"/' \
  /config/.storage/core.config_entries
```

Then restart HA. Never edit `.storage` files while HA is running (risk of corruption on save).

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
