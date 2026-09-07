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

Even for dashboards, though, prefer the WebSocket API (`lovelace/dashboards/create` + `lovelace/config/save`, see below). It needs no file access at all, takes effect immediately without a reload, and works when you are off the LAN.

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

### Reaching HA from outside the LAN

The same long-lived token works against the Nabu Casa remote URL
(`https://<id>.ui.nabu.casa`), which is the only option when the machine you
are working from is not on the home network — Samba, the local IP, and any
add-on web UI are all unreachable then.

Budget for latency: the first call over the relay can take 15 s or more.
Timeouts that are fine locally (10–20 s) will fail, and a failure looks
identical to the service being down. Use 60–120 s.

If local access suddenly stops working entirely — ping succeeds but every
port is closed, Samba is gone — check whether you are still on the same
network before diagnosing the HA host.

---

## WebSocket API

Some things the REST API cannot do at all. Connect to
`wss://<host>/api/websocket`, expect `auth_required`, send
`{"type": "auth", "access_token": ...}`, expect `auth_ok`, then send
commands with a monotonically increasing `id` and match responses by that id.

**Use one persistent connection for a sequence of calls.** Opening a fresh
connection per call is measurably unreliable over a remote relay — calls fail
sporadically with connection errors that look like permission problems.

### Creating dashboards without file access

This is the remote-friendly alternative to writing `.storage/lovelace.<id>`:

```json
{"type": "lovelace/dashboards/list"}
{"type": "lovelace/dashboards/create", "url_path": "my-dash", "title": "My Dash",
 "icon": "mdi:gauge", "show_in_sidebar": true, "require_admin": false}
{"type": "lovelace/config/save", "url_path": "my-dash", "config": {"views": [...]}}
{"type": "lovelace/config", "url_path": "my-dash"}
```

`lovelace/config` also reads a dashboard back — useful for exporting a
UI-built dashboard into version control.

Beware the default "Overview" dashboard: `lovelace/config` returns
`config_not_found` for it. That is not an error — it means the dashboard is
in auto-generated (strategy) mode. Writing a config to it is equivalent to
"take control" in the UI and **permanently ends the auto-generation**, so
every future device and area has to be added by hand. Never do this to add a
card; create a separate dashboard instead.

`url_path` must contain a hyphen — `lovelace/dashboards/create` rejects a
single word like `"mining"` with `invalid_format`. `"all-mining"` works.

### Consolidating dashboards

Merging several dashboards into one — each becomes a view (tab) — is safer as
a hide than a delete:

```json
{"type": "lovelace/dashboards/update", "dashboard_id": "avalon_mining",
 "show_in_sidebar": false}
```

This removes it from the sidebar without touching its stored config or URL —
reversible with the same call and `true`, and nothing that still links to the
old URL breaks. Fetch each source dashboard's views with `lovelace/config`,
give every view a **unique `path` and `title`** (collisions are likely: two
dashboards both having a view titled "Monitoring" is common), concatenate them
into one `views` array, and `lovelace/config/save` it under a new dashboard.
The card content within each view can be copied unchanged.

### Reordering the sidebar

Sidebar order and hidden-panel state are **synced per-user** through
`frontend/user_data`, under the single compound key `"sidebar"` — not the more
guessable `"sidebar-panel-order"` or `"sidebarPanelOrder"` (those keys accept
writes without error, which makes the mistake easy to miss: the call succeeds
and reads back exactly what you wrote, but the frontend never looks at it, so
the sidebar visibly doesn't change).

```json
{"type": "frontend/set_user_data", "key": "sidebar",
 "value": {"panelOrder": ["lovelace", "attersee-steuering", "energy", "map", "..."],
           "hiddenPanels": []}}
```

Panel identifiers are the `url_path` from `get_panels` (`{"type": "get_panels"}`,
not `frontend/get_panels`) for custom dashboards and built-ins alike — `energy`,
`logbook`, `history`, `map`, a custom dashboard's own `url_path`. Include every
panel you don't want reordered too, in its existing relative position, or it
may end up sorted unpredictably relative to the ones you did specify.

The one exception: the default Overview dashboard's `url_path` really is
`lovelace`, but ordering it under that key did not take — it kept sorting to
the end regardless of position in the list. This may be the historical `states`
key (Home Assistant's dashboard system was called "States" before Lovelace)
still governing that one entry as a back-compat quirk; adding `"states"` to
the list did not resolve it either in the version tested here. Unconfirmed —
treat Overview's position as not reliably controllable via this API for now,
and don't spend much time chasing it if the rest of the order is right.

Takes effect on the next full page reload — no HA restart needed, since it's
pure per-user frontend state, not server config.

### Supervisor and add-ons

Add-on management goes through `supervisor/api`:

```json
{"type": "supervisor/api", "endpoint": "/supervisor/info", "method": "get"}
{"type": "supervisor/api", "endpoint": "/store/repositories", "method": "post",
 "data": {"repository": "https://github.com/owner/repo"}}
{"type": "supervisor/api", "endpoint": "/store/addons/<slug>/install", "method": "post"}
{"type": "supervisor/api", "endpoint": "/addons/<slug>/start", "method": "post"}
{"type": "supervisor/api", "endpoint": "/addons/<slug>/options", "method": "post",
 "data": {"boot": "manual"}}
```

Useful read endpoints: `/addons` (installed, as `{"addons": [...]}`),
`/store/addons` (everything available), `/addons/<slug>/info`,
`/store/addons/<slug>` (has `available`, `arch`, `installed`, `version`).

Three traps, each of which cost real time:

- **Never send a `timeout` field.** With it, every call fails as
  `{"code": "unknown_error", "message": ""}` regardless of the endpoint —
  which reads exactly like a permission problem and sends you off
  investigating admin rights and allowlists. Without it, the same calls work.
- **Long operations report failure and succeed anyway.** An add-on install
  downloads a container image and outlives the response window, returning the
  same empty `unknown_error`. The install completes regardless. Always
  re-query `/addons` or `/store/addons/<slug>` afterwards instead of trusting
  the error — otherwise you will report a failure that did not happen, or
  leave something installed without noticing.
- **Match add-on slugs exactly.** A substring match on `"uni"` hits
  `a0d7b954_unifi` before `663b81ce_uni_meter`, and combined with the
  previous trap you can install the wrong add-on and be told it failed.

### Add-on logs

Logs are plain text, so `supervisor/api` cannot return them. The REST proxy
does, and this specific path works with a long-lived token even though most
`/api/hassio/*` paths return 401:

```bash
curl -s -H "Authorization: Bearer $TOKEN" -H "Accept: text/plain" \
  "$HA/api/hassio/addons/<slug>/logs" | sed 's/\x1b\[[0-9;]*m//g'
```

The `sed` strips ANSI colour codes. For an add-on that talks to an external
service, several minutes of log with no connection errors is decent evidence
that its credentials and URLs are right.

### Reading the error log

`GET /api/error_log` is gone in recent versions (404). Use the WebSocket
command `system_log/list` instead — it returns structured entries with
`level`, `name`, `message`, `count` and `exception`, which is more useful than
raw log text anyway:

```json
{"type": "system_log/list"}
```

The `count` field is the killer feature: an entry seen 6216 times is a loop,
one seen twice is a blip. Filter by the integration's `name`
(`custom_components.<domain>`) to isolate one device.

---

## Energy Dashboard

Configured over WebSocket, not YAML:

```json
{"type": "energy/get_prefs"}
{"type": "energy/save_prefs", "energy_sources": [...], "device_consumption": [...]}
```

`save_prefs` takes the fields at the top level of the message, not nested — read
the current prefs, modify, and send the whole structure back.

Structure:

```json
{
  "energy_sources": [
    {"type": "solar",  "stat_energy_from": "sensor.pv_production"},
    {"type": "grid",
     "flow_from": [{"stat_energy_from": "sensor.grid_import"}],
     "flow_to":   [{"stat_energy_to":   "sensor.grid_export"}]},
    {"type": "battery",
     "stat_energy_from": "sensor.battery_out",
     "stat_energy_to":   "sensor.battery_in"}
  ],
  "device_consumption": [
    {"stat_consumption": "sensor.ev_charger_total", "name": "EV charger"}
  ]
}
```

Every entity here must be **energy** (kWh/Wh) with a `state_class` of `total`
or `total_increasing` — power sensors are rejected. For a device that only
reports watts, create a Riemann-sum helper first (below).

Two things worth knowing:

- **Devices under `device_consumption` are informational.** They are shown as a
  breakdown of consumption, not subtracted from it — so listing a device twice,
  or listing a *production* counter as a device, silently skews the picture.
  Check what a candidate sensor actually measures: a counter named
  `<inverter name>_total_energy` is usually lifetime PV yield, not consumption.
- **A battery declared here fixes the accounting automatically.** HA then knows
  charging is storage rather than consumption and draws the flow diagram
  correctly — which is the supported alternative to the manual correction in
  the next section.

### Creating helpers over the API

Helpers that have a config flow (`integration`, `template`, `utility_meter`,
`derivative`, `threshold`, `min_max`, …) can be created entirely over REST — no
file access, no restart:

```bash
# 1. Start the flow, read the returned schema
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" \
  -d '{"handler":"integration","show_advanced_options":true}' \
  "$HA/api/config/config_entries/flow"

# 2. Answer it (fields exactly as the schema names them)
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" \
  -d '{"name":"House consumption energy","source":"sensor.house_power",
       "unit_prefix":"k","unit_time":"h","method":"trapezoidal",
       "round":3,"max_sub_interval":{"minutes":2}}' \
  "$HA/api/config/config_entries/flow/$FLOW_ID"
```

`max_sub_interval` matters for a Riemann sum: without it, a sensor that stops
updating (because its value is unchanged) contributes nothing, and the integral
silently stalls.

**Do not assume the resulting entity_id.** HA prefixes helper entities with the
source entity's device and area, so `name: "House consumption energy"` can land
as `sensor.<device>_<area>_house_consumption_energy`. Look it up afterwards by
searching states for the friendly name rather than guessing.

---

## Battery Behind the Meter

A trap that produces plausible-looking but wrong numbers, on any system where a
battery is added later.

A grid meter at the connection point measures **import and export directly** —
those stay correct no matter what sits behind it. But "house consumption" is
usually not measured at all, it is *computed*:

```
load = pv_production + grid_import − grid_export
```

Add a battery behind the meter and that formula breaks:

| Situation | What the formula yields | Why it is wrong |
|---|---|---|
| Battery charges at 1200 W | load +1200 W | storing is not consuming |
| Battery discharges at 800 W | load −800 W | consumption understated |

The vendor's own app and cloud portal make exactly the same mistake, for the
same reason — so "my dashboard disagrees with the manufacturer's app" is not
proof that the dashboard is wrong once a battery is in play.

The fix needs the battery to report its own flows (most do, over MQTT or a
local API):

```yaml
- name: "House consumption (corrected)"
  unit_of_measurement: "W"
  device_class: power
  state_class: measurement
  availability: "{{ states('sensor.computed_load') | is_number }}"
  state: >
    {% set load = states('sensor.computed_load') | float(0) %}
    {% set net  = states('sensor.battery_charge_power') | float(0)
                - states('sensor.battery_discharge_power') | float(0) %}
    {{ [load - net, 0] | max | round(1) }}
```

Two practical notes:

- **Prepare it before the battery arrives.** Define the two battery terms as
  template sensors returning `0`. Everything downstream then reads correctly
  today and becomes correct automatically once you point those two at the real
  entities and call `template.reload` — no restart, no rework.
- **Sanity-check the direction.** While charging, the corrected value must be
  *lower* than the raw computed load. If it is higher, charge and discharge are
  swapped — an easy mistake when an integration reports a single signed value.

Also worth capturing at the same time, since the inputs are already there:
self-sufficiency (`1 − grid_import / load`) and self-consumption
(`1 − grid_export / pv`). Clamp both to 0–100 and guard the division.

### Comparing against the manufacturer's app

Compare **energy totals, never instantaneous power.** Two systems poll at
different moments, and a house load swings by hundreds of watts as fridges,
chargers or miners cycle — a snapshot comparison will show a 350 W "error" that
does not exist. Daily kWh counters, by contrast, should agree to the second
decimal; if they do, the setup is fine.

Expect a small permanent residual in `pv − load − export`, typically tens of
watts. That is the inverter's own consumption and measurement tolerance between
inverter and meter, present in the vendor's raw data — not something to correct.

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
- **HACS updates overwrite custom component files.** Keep patched files in your own repo alongside a note on how to redeploy, or the patch will vanish at the next update with no warning. The symptom is indirect: the feature you added is simply missing again, with nothing in the logs.

### Re-applying a patch after an update

Do not just copy your saved file back — that would silently revert whatever the
update changed. Reconstruct instead:

1. Fetch the new upstream file (`raw.githubusercontent.com/<owner>/<repo>/main/…`;
   find the repo via HACS `hacs/repositories/list`).
2. Strip your additions from the saved copy to recover the base you patched.
3. Diff that base against the new upstream. **Often it is empty** — the update
   touched other files entirely, and your patch re-applies verbatim.
4. Re-apply, syntax-check (`ast.parse`), redeploy, restart.

If this recurs, stop patching in place: **fork the repo, apply the patch there,
and add the fork as a HACS custom repository.** Updates then come from the fork
and the patch is part of the package. A pull request upstream is worth sending
too — if it lands, the fork becomes unnecessary.

**Redeploying "step 4" means copying to the *live* instance, not just updating
your saved copy.** It is easy to regenerate the patched file, verify it locally,
commit it to your own repo for safekeeping — and stop there, having never
touched `/config/custom_components/...` on the actual running system. The
symptom is silent and delayed: the feature works for a while (the last time it
really was deployed), then mysteriously "stops working after a restart" once
something else (an update, a reinstall) later wipes the live file back to
upstream — even though your repo has looked correct the whole time. Before
concluding a patch problem is anything more exotic, `diff` your saved copy
against the file that is actually on the instance. An empty diff means it is
truly deployed; any diff means step 4 didn't happen yet.

---

## Entity Registry: Renaming Instead of Rewiring

When a device is deleted and re-added (rather than reconfigured in place — see
"Patching a Custom Component" above for why that matters), Home Assistant
creates a **new** device with a new `unique_id`, and entity IDs are regenerated
from scratch. If the new IDs differ from the old ones — a different area
assigned during setup changes the suggested prefix, for instance — every
dashboard and automation that hardcoded the old entity_id breaks at once,
showing "Entity not found."

The tempting fix is to edit every dashboard card and automation reference to
the new names. Do not — there is a better tool for exactly this situation:
**rename the entities in the registry back to their old IDs.** This touches
nothing else; dashboards and automations keep working unmodified, because they
only ever referenced the entity_id string, not the device.

```json
{"type": "config/entity_registry/list"}
```

Filter the result by `device_id` (from `config/device_registry/list`) or by
`config_entry_id` to get exactly the entities belonging to the recreated
device, then rename each:

```json
{"type": "config/entity_registry/update",
 "entity_id": "sensor.keller_avalon_nano_3s_2_hashrate",
 "new_entity_id": "sensor.miner_avalon_nano_3s_2_hashrate"}
```

Renames take effect immediately, no restart needed, and survive a restart once
applied. Two things to check first:

- **No orphaned entities are already sitting on the target names.** List the
  full registry and grep for the old prefix before renaming — if the deleted
  device's entries linger (they normally don't once the config entry is
  removed), the rename will collide.
- **The integration's own naming may have drifted independently.** A device
  that was first set up long ago keeps whatever key names its sensors had
  *at that time* ("grandfathered"), even if the same integration version
  would generate different keys for a brand-new device today (a sensor
  renamed upstream between then and now, still using the old key on the old
  device). Compare the freshly-recreated device's entities against an
  untouched sibling device on the same integration to catch this — some
  fields may need remapping to a differently-named field with the same
  meaning, and a few may have no equivalent left at all (in which case fix
  the *reference*, e.g. point a dashboard template at the closest surviving
  field, rather than renaming something that doesn't exist).

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

### Automations without an explicit `id:` spawn duplicate entities

An automation entity_id is normally derived once from its `alias` and then
persisted in the entity registry — but only if the automation has an explicit
`id:` field pinning it to that registry entry. Without one, a reload can
regenerate a new internal id, and the entity registry then has to reconcile a
"new" automation with the same alias-derived name: it keeps the old entry
(now orphaned, permanently `unavailable`, nothing in YAML feeds it anymore)
and creates a second one with a `_2` suffix for the one that's actually live.
Repeat reloads compound this — `_3`, `_4`, and so on, cluttering the registry
and, worse, making it easy to edit the dead entry in the UI and wonder why
nothing happens.

Give every automation an explicit `id:` (any stable string, commonly a
timestamp) to prevent this outright — YAML-authored automations especially,
since GUI-created ones get one automatically:

```yaml
- id: "1788685815"
  alias: My Automation
  ...
```

If duplicates already exist, find the live one by state (`on`, matching the
YAML) versus the orphan (`unavailable`), then remove the orphan via
`config/entity_registry/remove` and add `id:` going forward so it can't recur.

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

### "Add-ons" are called "Apps" in recent versions

The UI section is **Settings → Apps**, and on disk `/addon_configs` shows up
as `app_configs`. Instructions written against the old naming send people
looking for a menu entry that no longer exists. Ask what the user actually
sees rather than insisting on the documented name — upstream add-on
documentation still says "add-on" throughout.

### Add-on config files that the add-on does not create

Several add-ons expect a config file in `/addon_configs/<slug>/` that they
will not create themselves, and the directory only appears once the add-on
has started at least once. Starting it once to have Supervisor create the
directory with correct ownership is more reliable than creating it by hand.

Writing that file needs Samba (LAN only) or the **File editor** add-on — and
File editor ships with `enforce_basepath: true`, which confines it to
`/config`. Turning that off in the add-on's configuration and restarting it
exposes the whole filesystem; turn it back on afterwards.

### `footer:` on an entities card breaks it in the sections layout

An `entities` card with a `footer:` renders a red **Configuration error** box below the entity rows when used in a `type: sections` view:

```yaml
# Broken in a sections view
- type: entities
  title: Solar Control
  entities:
    - entity: input_boolean.solar_control
  footer:
    type: markdown
    content: "**On**: automatic. **Off**: manual."
```

Use a separate `markdown` card in the same grid section instead:

```yaml
- type: entities
  title: Solar Control
  entities:
    - entity: input_boolean.solar_control
- type: markdown
  content: |
    **On**: automatic.

    **Off**: manual.
```

The failure is easy to misdiagnose, because the entity rows above the error render normally — it looks like a broken entity rather than a broken card option.

### A `loaded` config entry with dead entities

`state: "loaded"` only means setup succeeded once. If every entity of that
entry reads `unavailable` anyway, the coordinator is failing its updates — the
entry state will not tell you. `system_log/list` will:

```
Connection failed for command 'version' after 3 attempts:
[Errno 113] Connect call failed ('192.168.1.50', 4028)    count: 6216
```

Read the errno rather than guessing. `113` (EHOSTUNREACH) means nothing
answered at that address — the device is off or gone, which is *not* the same
as a wrong IP. `111` (ECONNREFUSED) means the host is up but that port is
closed, which points at a service or port problem instead.

Note the port in the message: an integration often talks on a different port
than the device's web UI, so "I can open it in the browser" does not prove the
integration's path works.

When a device comes back after being offline, the coordinator does not always
recover on its own. Reloading the entry is enough — no restart:

```bash
curl -s -X POST "${AUTH[@]}" \
  "$HA/api/config/config_entries/entry/$ENTRY_ID/reload"
```

If a device is switched on and off routinely (a solar-controlled socket, say),
automate that reload on the switch turning on rather than fixing it by hand.

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

## Emulating a Smart Meter for a Battery/Inverter Integration

Some battery systems only accept surplus/consumption data from a smart meter
they can discover themselves — no generic "push power reading here" API.
A local add-on that reads real sensor values from HA and re-serves them as an
emulated meter (mirroring the protocol/API of a specific meter model) bridges
this. The emulation target matters more than it looks:

- **Don't assume any emulated protocol will be accepted just because it's on
  the device's compatibility list.** A device may claim support for meter
  family X while only ever having been tested against real X hardware —
  the app finds an emulated X but the device itself never starts polling it.
  Check community issue trackers for the *specific* device model before
  picking which protocol to emulate; a working combination reported by
  someone else with the same device is worth more than the general
  compatibility list.
- **mDNS/zeroconf discovery is often mandatory, not optional**, when the
  emulated meter is one that ships with self-announcing hardware. Without an
  explicit mDNS announcement (e.g. via a pyscript service that registers the
  service record), the companion app or the device itself never finds the
  emulated endpoint at all — this fails silently as "just doesn't show up",
  not as an error.
- **Never keep default/example identity values (MAC, serial) from the
  emulator's documentation.** If multiple installations copy the same
  example values, they collide on the discovery network and connections
  drop unpredictably whenever another instance of the same setup appears —
  intermittent and hard to attribute. Generate your own locally-unique
  values.
- **A found-but-never-connects device often means the identity values look
  synthetic, not that transport is broken.** If the app *finds* the emulated
  meter via discovery but the target device reports it "offline" — while
  network reachability, mDNS query/answer exchange, and even unicast
  responses all check out — suspect the identity fields themselves. Some
  devices silently discard a discovery answer whose vendor ID fields don't
  resemble their real counterparts (e.g. a MAC using an actual hardware
  vendor's OUI prefix instead of a fully custom locally-administered
  address, or a serial number in the vendor's specific format rather than a
  simple reused string). Before concluding transport is broken, exhaust the
  cheap checks in order: TCP reachability, discovery packets actually being
  exchanged (packet capture inside the automation platform if you can't run
  one on the network itself), then only after those pass, try identity
  values that mimic a real device's format instead of arbitrary ones.
  Confirm with a protocol-level packet capture (a small script that binds
  the discovery port and counts senders/queries) before concluding the
  answer isn't reaching the device — that isolates "my answer never
  arrived" from "my answer arrived but was rejected".
- **After changing the emulated identity, the pairing must usually be redone
  from scratch on the companion app/device side**, not just restarted on the
  HA side — many pairing flows have the app derive a device ID once from the
  discovery record and hand it to the target device over a side channel
  (e.g. Bluetooth) at pairing time; changing the identity afterwards doesn't
  retroactively update what the device is looking for.
- **A self-announcing service registered via a scripting layer (not the
  add-on itself) is usually lost on every restart of the automation
  platform**, because the in-memory discovery registry is cleared while the
  add-on only re-announces at its own startup. If a restart of the add-on
  alone then fails with a duplicate-name error, the stale registration is
  still sitting in the platform's registry from before the platform
  restarted — clear it explicitly (an unregister/cleanup call) before
  restarting the add-on. This is easy to mistake for "the identity change
  didn't take" because the symptom (still using the old identity) looks
  identical; automate the cleanup-then-restart sequence to trigger on
  platform startup so it isn't a manual step every time.
- Turn on protocol-level (HTTP/TCP) trace logging on the add-on only while
  debugging pairing — it reveals exactly which client IP is polling which
  endpoint — and revert it to the normal level afterwards; left on, it is
  noisy and fills logs fast.
- **A polled MQTT/local integration can restore real-time fields (power,
  status) immediately after a restart while cumulative counters lag by
  many minutes.** If the device only includes its cumulative energy/energy
  totals in a less-frequent or larger "full report" message type (distinct
  from the frequent lightweight status ticks), those specific entities sit
  at `unknown` for a noticeably longer window after every restart than the
  rest of the device's entities — don't mistake that gap for a broken
  pairing when the power/status fields already look healthy; give the
  cumulative fields their own, longer grace period before troubleshooting.
  This has a knock-on effect on the Energy dashboard: its "current, still
  open hour" tile is computed as *completed-hours sum + (live value − value
  recorded exactly at the start of this hour)*. If the entity's value was
  `unknown` at the moment the hour rolled over (a likely restart timing
  coincidence), that reference point doesn't exist, and the tile shows 0
  for the entire remainder of that hour even though the underlying sensor
  is already reporting correctly — it self-resolves once the next full
  hour completes with a continuously-valid value throughout. Confirmed in
  practice: the tile stayed at 0 for the rest of the hour the restart fell
  in, then showed the correct accumulated value as soon as the following
  hour's statistic compiled — no manual fix, config change, or further
  restart needed. Don't conclude the Energy dashboard configuration is
  wrong from a stuck-at-zero tile alone; check the entity's raw state and
  the recorder's short-term (5-minute) statistics first — if those already
  show real, growing numbers, the dashboard tile is just waiting out this
  hour.

### A follow-meter control loop that oscillates: check the update rate before the math

A battery/inverter following an emulated meter can end up cycling between
charging and discharging (or feed-in and draw) every 60-90 seconds instead
of settling. Before suspecting the meter *value* is wrong, check the meter
*update rate* against the device's own polling rate. If the device polls
the emulated meter every few seconds but the value only refreshes on a
much slower cadence (e.g. a cloud-backed integration polling its source
once a minute), the device re-adjusts many times before it ever sees the
effect of its previous adjustment — a control loop with that much dead
time will oscillate almost by construction, regardless of how correct the
underlying formula is. The fix is to shorten the path: read the true
source directly and frequently (bypassing the slower intermediate
integration) rather than manipulating the value that gets fed through.

**A tempting but wrong fix to rule out first:** if the emulated meter sits
between the true meter and the device, and the device's own charge/
discharge action visibly shows up in that meter's reading (because the
device is wired behind it), it's tempting to "correct" the fed value by
subtracting the device's own contribution — reasoning that the device is
being fed its own delayed action and chasing its own shadow. Resist this
without first testing the update-rate fix: subtracting the device's own
effect from what it's told doesn't just remove a feedback artifact, it
removes the negative feedback the control loop actually needs to
self-limit. The oscillation may well disappear — but only because the
signal now always reads "there is demand" whenever the device is doing
*anything*, so it locks onto its own output ceiling and stays there
regardless of the real, changing demand, silently overproducing (in a
battery/grid context: wasting the excess as involuntary export). A stable-
looking result is not proof the fix is right; verify against a ground-
truth reading (e.g., put the device in standby and read the true meter
directly) that the corrected value tracked reality *before* trusting it,
and specifically test the case where real demand drops below the device's
configured power ceiling — that is exactly the regime the original
oscillation happened in, and the regime a rate fix (unlike a value
"correction") should fix cleanly.
