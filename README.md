

# Note
- Solax X1-G4 Inverter (may not work with other inverters as settings and modes may be in different read/set positions)
- For reading live data you'll need this [repo](https://github.com/RGx01/Solax-Local-Realtime-Using-REST) or try [dashboard repo](https://github.com/RGx01/home-assistant-Solax-Zappi-Octopus-Control)

# Install Instructions
1. In your Home Assistant /config directory create two directories (if they don't already exist)
   * packages
   * scripts
2. Copy repo package directory to your package directory
3. Edit secrets.yaml to replace YYYYYYYYYY with your registration number (found on the devices page on the solax cloud)
4. Copy repo script directory to your scripts directory
5. Edit your configuration.yaml file (stored in the /config directory) to include:
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
Sets forced charge start time
```yaml
action: script.solax_set_mode_and_settings
data:
  settings:
    forced_charge_start: >-
      {% set start_time =
      as_datetime(state_attr('input_datetime.solax_battery_start_charge_time',
      'timestamp') - 0) %}{{ start_time.hour + start_time.minute * 256 }}
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
###### An untested automation example of how you can get a persistant notification from the script.
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
Battery heating settings can't be changed as the dongle is locked to basic settings (for now).

# Revision Log
| Version | Date | Files updated |Description |
|:------|:--------:|:------|:------|
| v2.0.1 | **05/12/25**| templates.yaml <br> scripts\solax_set_mode_and_settings.yaml <br> sensor.yaml| Minor tweak to templates to remove measurement tag from some entities also forgot to consolidate comments on versions of the yaml in the other files (no code changed)|
| v2.0.0 | **13/11/25**| scripts\solax_set_mode_and_settings.yaml <br> sensor.yaml| Improved code effieciency so that only relevant changes are applied <br> sensor timeout increased|
| v1.3.6 | **13/10/25**| scripts\solax_set_mode_and_settings.yaml| Fixed timeout syntax|
| v1.3.5 | **12/10/25**|scripts\solax_set_mode_and_settings.yaml| Bug fix to prevent premature exit after 1 try if theres a failure setting settings or modes|
| v1.3.4 | **11/10/25** |scripts\solax_set_mode_and_settings.yaml| Bug fix with success and needs_update logic|
| v1.3.3 | **11/10/25**|scripts\solax_set_mode_and_settings.yaml| Bug fix with success and needs_update logic |
| v1.3.2 | **11/10/25**|scripts\solax_set_mode_and_settings.yaml| Properly fixed bug in for loops that would prevent retrying if settings were not applied correctly |
| v1.3.1 | **11/10/25**|scripts\solax_set_mode_and_settings.yaml| fixed bug in for loops that would prevent retrying if settings were not applied correctly |
| v1.3.0 |**7/10/25**|scripts\solax_set_mode_and_settings.yaml <br>templates.yaml<br>sensor.yaml| Increased efficiency of script, set forced update and no cache options, This increases the reliability of the check and confirmation of settings. I discovered that in various circumstances the rest command can return cached responses or ghost responses due to network issues. hopefully now we should get better responses and more effective retries.|
| v1.2.1 |**21/09/25**| scripts\solax_set_mode_and_settings.yaml | Updated rest commands for battery heating (not really needed as they can't be set anyway) |
| v1.2.0  | **13/09/25** | templates.yaml | Tidy up |
| v1.1.0  | **13/09/25** | All | Added template sensors so they can be viewed in HA <br> removed realtime rest call and secret|
| v1.0.0  | **12/09/25** | All | Major refactor |
