# Solax-Local-Control-Using-REST
# Equipment Used During Development
<ol>
<li>Solax X1-G4 Inverter (may not work with other inverters)</li>
<li>TP3.0 - HV10230 Batteries</li>
</ol>

# Install Instructions
1. In your HA /config directory create two directories (if they don't already exist)
   * packages
   * scripts
2. Copy repo package directory to your package directory
3. Copy repo script to your scripts directory
4. Edit your configuration.yaml file (stored in the /config directory) to include:
```yaml
homeassistant:
  packages: !include_dir_merge_named packages/
script: !include_dir_merge_named scripts
```
6. Restart HA

# Automation Usage Examples

Turns off period 2, sets the Self Use charge to limit and sets the self use min Soc to 15
```yaml
action: script.solax_set_mode_and_settings
data:
  mode: 0
  settings:
    period2_enabled: 0
    selfuse_charge_battery_from_grid: 100
    selfuse_battery_min_soc: 15
```

Starts a force discharge
```yaml
action: script.solax_set_mode_and_settings
data:
  mode: 3
  manual_mode: 2
```


Use other states to control modes and settings
```yaml
action: script.solax_set_mode_and_settings
data:
  mode: "{{states('sensor.solax_default_operation_mode')|int}}"
  settings:
    period2_enabled: "{{0}}"
    selfuse_charge_battery_from_grid: "{{states('input_number.solax_default_charge_to_limit_soc')|int}}"
    selfuse_battery_min_soc: "{{states('input_number.solax_default_discharge_limit_soc')|int}}"
```
# Script Configuring 
## Delays
In solax_set_mode_and_settings.yaml, in the variable section refresh_delay and settings delay and mode_delays can be set. The refresh delay is used to make sure the latest setting on the inverter are read correctly, settings delay is used to ensure settings i.e. min_soc get updated on the inverter. Mode delays are longer and can be different for various modes as the inverter takes longer to change modes. Note that mode 3 (manual mode) delay is also used when setting the manual_mode function (0 do nothing, 1 force charge, 2 force discharge)
```yaml
    - variables:
        refresh_delay: 5
        settings_delay: 5
        mode_delays:
          0: 20
          1: 20
          3: 25
```
## loops
Max number of loops can be adjusted, if you need more than 5 something is wrong!
```yaml
    - variables:
        max_loops: 5
```
# Script Notifications
The script can generate a notification event but you will need an automation to listen for those events. an example (untested)
```yaml
alias: Notifier
description: >-

  Allows for notification of activity

  v1.0 Initial
triggers:
  - trigger: event
    event_type: Solax Control
    event_data:
      class: normal
    id: Normal
  - trigger: event
    event_type: Solax Control
    event_data:
      class: high
    id: High
conditions: []
actions:
  - action: logbook.log
    data:
      name: Solax Zappi Octopus
      message: |
        Event ({{ trigger.event.data.class }}): {{ trigger.event.data.title }}
  - choose:
      - conditions:
          - condition: or
            conditions:
              - condition: trigger
                id:
                  - Normal
              - condition: trigger
                id:
                  - High
        sequence:
          - action: persistent_notification.create
            data:
              title: "{{trigger.event.data.title}}"
              message: "{{trigger.event.data.message}}"
mode: queued
max: 10

```
# Script Requirements
Requirements: solax_set_mode_and_settings
1. Inputs
mode (optional, integer)
Valid values: 0, 1, 3
Represents inverter operating mode:
0 = Self Use
1 = Feed In Priority
3 = Manual
manual_mode (optional, integer)
Valid values: 0, 1, 2
Only relevant if mode = 3 (Manual):
0 = Do Nothing
1 = Force Charge
2 = Force Discharge
settings (optional, dict)
Key-value pairs mapping Solax inverter setting names to desired values.
Keys must be present in the idx_map.
Values are assumed to be integers.
2. Preconditions
The script must update the Solax entity (sensor.solax_rest_local_settings) before starting, to ensure current values are fresh.
If no changes are required (needs_update = false), the script should exit early and log "No changes required.".
3. Behavior
The script determines whether changes are required by comparing:
Current mode vs. requested mode.
Current manual_mode vs. requested manual_mode (if applicable).
Current values of each requested settings key vs. target values.
If any mismatch is detected (needs_update = true), the script attempts to apply changes.
Changes are retried in a loop:
Maximum max_loops attempts (default = 5).
Stops earlier if all requested values are successfully applied.
Each loop delays by refresh_delay seconds after an update.
Timeout enforced (timeout_seconds = 300).
4. Mode handling
If a new mode is requested:
For mode != 3:
Reset manual_mode to 0.
Apply the new mode.
For mode = 3:
Apply Manual mode.
If manual_mode is provided, apply it.
Each mode change includes an additional mode_delay wait, which depends on mode.
5. Settings handling
For each key-value pair in settings:
Look up service and field from idx_map.
Call the corresponding rest_command with the correct payload field.
Wait settings_delay between setting changes.
6. Validation
After each loop, validate success:
mode matches requested mode.
manual_mode matches requested value (if applicable).
All requested settings match the target values.
If validation passes, mark success = true and stop looping.
7. Failure handling
If success is still false after max_loops or timeout_seconds:
Fire a Solax Zappi Octopus Control event with:
message: Description of which settings failed.
title: "Solax Set Failure - Action required!".
class: "high".
8. Logging
If no changes were required, log a logbook.log entry with:
name: "Solax Set Mode and Settings"
message: "No changes required. Mode=X, Manual=Y, Settings=Z"

# Notes
Battery settings can't be changed as the dongle is locked to basic settings (for now).

