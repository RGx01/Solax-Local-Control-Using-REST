# Solax-Local-Control-Using-REST
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

Turns off period 2m sets the Self Use charge to limit to 100 and min Soc to 15
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

# Notes
Battery settings can't be changed as the dongle is locked to basic settings (for now).
## Equipment Used During Development
<ol>
<li>Solax X1-G4 Inverter</li>
<li>TP3.0 - HV10230 Batteries</li>

