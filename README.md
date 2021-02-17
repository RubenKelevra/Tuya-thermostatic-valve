# Tuya-thermostatic-valve
automation documentation for better temperture control, with an external temperature probe / window contact


# prerequisites

- Home Assistant (2021.2.0 at the time of writing)
- Properly setup Zigbee2MQTT plugin (or similar)
- properly installed
  - [TuYa TS0601](https://www.zigbee2mqtt.io/devices/TS0601_thermostat.html) thermostat
  - [Aqara window contact sensor](https://www.zigbee2mqtt.io/devices/MCCGQ11LM.html) (or similar)
  - [TuYa TS0121 plug](https://www.zigbee2mqtt.io/devices/TS0121_plug.html) (or similar) for any electrical heat source
- latest thermostat firmware
- thermostat settings:
  - Make sure the build-in "window detection" is off
  - Set minimal temperature to 1°C
  - Set maximum temperature to something high, like 35°C
  - Set hysteresis to 0.5°C (default is 1.0°C)
  - Make sure the valve control type is set to "PID"
  - there's no need to set any 'local temperature correction' - this value won't be touched, but has no effect on the automation.
- Enable the OpenWeatherMap integration(or similar) to get an outside temperature

# recommended mods

- Attach a 3 V power supply at the pcb to avoid fast battery drainage due to larger/more often valve operation, see [power supply mod](power_supply_mod.md)

# limitations

- Can only set one temperature, if you need a week/daily cycle, you need to modify the automations or just modify the slider
- Can only work with an good external temperture probe, which is properly installed, see [installation temperature probe](installation_temperature_probe.md) for help
- there's no "get the heat on fast" for a very cold room (but the TRV should turn on the heat pretty quickly anyway, since there's a high temperature differential)
- there's no "boost mode" if you feel very cold and just want a short increase in temperature. *Workaround:* feel free to use just your own automation for this, to create a button which increases the target temperature for a while and then turn it back to the previous setting. You can also still use the boost mode on the TRVs
- The controls at the TRV are completely useless. Every change there will be overwritten on the next change by the automation 

# helpers

Create the following helpers:

<details>
  
- `input_select.heating_mode_kitchen` with 3 values: `default`, `off` and `cut`
- `input_boolean.heatsource_present_kitchen`
- `input_boolean.heating_maintaince_kitchen`
- `input_select.heating_forced_off_weather` with 3 values: `no`, `no-heating-required`, `hot-day`
- `input_boolean.heating_kitchen`
- `input_number.target_temp_kitchen` with step size 0.5 and ℃ as unit

</details>

Warning: Only manually modify the last two manually, the rest is to store the status of the automations.

# expected naming of sensors/actuators

- `climate.radiator_valve_kitchen` for the thermostat
- `sensor.power_plug_airfryer_power` and `sensor.power_plug_exhaust_hood_power` for heat source detection
- `sensor.temp_sensor_kitchen_temperature` for the temperature probe
- `binary_sensor.window_sensor_kitchen_contact` for the window sensor
- `sensor.openweathermap_temperature` for an environment temperature sensor

# statistics sensors

- `sensor.derivation_filtered_temperature_kitchen`:
```yaml
sensor:
  - platform: derivative
    name: "Derivation filtered Temperature - Kitchen"
    source: sensor.kitchen_filtered_temperature
    time_window: "00:03:00"
```

- `sensor.kitchen_filtered_temperature`:
```yaml
sensor:
  - platform: filter
    name: "Kitchen filtered temperature"
    entity_id: sensor.temp_sensor_kitchen_temperature
    filters:
      - filter: time_simple_moving_average
        window_size: "00:01:00"
        precision: 2  
```

- `sensor.temperature_statistics_24h`
```yaml
sensor:
  - platform: statistics
    name: 'Temperature statistics 24h'
    entity_id: sensor.openweathermap_temperature
    max_age:
      hours: 24
    sampling_size: 86400
    precision: 2
```

# heat source detection

This automatisms is an example how to detect a running stove or a heat generateing device early, to avoid running the heat at the same timewhile those devices are running. This way you can avoid venting the room in some cases, when it would become too hot otherwise.

Detection with just the temperature probe is too delayed in most cases to help much.

<details>
  
```yaml
alias: 'Heating: Detect heat source - Kitchen'
description: ''
trigger:
  - type: power
    platform: device
    entity_id: sensor.power_plug_airfryer_power
    domain: sensor
    above: 100
  - type: power
    platform: device
    entity_id: sensor.power_plug_exhaust_hood_power
    domain: sensor
    above: 2
condition:
  - condition: or
    conditions:
      - type: is_power
        condition: device
        entity_id: sensor.power_plug_airfryer_power
        domain: sensor
        above: 100
      - type: is_power
        condition: device
        entity_id: sensor.power_plug_exhaust_hood_power
        domain: sensor
        above: 2
action:
  - service: input_boolean.turn_on
    data: {}
    entity_id: input_boolean.heatsource_present_kitchen
mode: queued
max: 10
```

```yaml
alias: 'Heating: Detect heat source gone - Kitchen'
description: ''
trigger:
  - type: power
    platform: device
    entity_id: sensor.power_plug_airfryer_power
    domain: sensor
    below: 100
    for:
      minutes: 5
  - type: power
    platform: device
    entity_id: sensor.power_plug_exhaust_hood_power
    domain: sensor
    below: 3
    for:
      minutes: 5
condition:
  - condition: and
    conditions:
      - type: is_power
        condition: device
        entity_id: sensor.power_plug_airfryer_power
        domain: sensor
        below: 100
      - type: is_power
        condition: device
        entity_id: sensor.power_plug_exhaust_hood_power
        domain: sensor
        below: 3
      - condition: state
        entity_id: input_boolean.heatsource_present_kitchen
        state: 'on'
action:
  - service: input_boolean.turn_off
    data: {}
    entity_id: input_boolean.heatsource_present_kitchen
mode: queued
max: 10
```

</details>

# heat mode automation

This automatisms shuts the valves in all scenarios where this is commanded by other automatisms or by the user.

<details>
  
```yaml
alias: 'Heating: Shut valve when heat mode is off/cut - Kitchen'
description: ''
trigger:
  - platform: state
    entity_id: input_select.heating_mode_kitchen
    to: 'off'
  - platform: homeassistant
    event: start
  - platform: state
    entity_id: input_select.heating_mode_kitchen
    to: cut
condition:
  - condition: or
    conditions:
      - condition: state
        entity_id: input_select.heating_mode_kitchen
        state: 'off'
      - condition: state
        entity_id: input_select.heating_mode_kitchen
        state: cut
action:
  - choose:
      - conditions:
          - condition: state
            entity_id: input_boolean.heating_maintaince_kitchen
            state: 'on'
        sequence:
          - wait_for_trigger:
              - platform: state
                entity_id: input_boolean.heating_maintaince_kitchen
                from: 'on'
                to: 'off'
            timeout: '00:15:00'
          - service: climate.set_hvac_mode
            data:
              hvac_mode: 'off'
            entity_id: climate.radiator_valve_kitchen
      - conditions:
          - condition: state
            entity_id: input_boolean.heating_maintaince_kitchen
            state: 'off'
        sequence:
          - service: climate.set_hvac_mode
            data:
              hvac_mode: 'off'
            entity_id: climate.radiator_valve_kitchen
    default: []
mode: queued
max: 10
```

```yaml
alias: 'Heating: Shut valve when heating is deactivated - Kitchen'
description: ''
trigger:
  - platform: state
    entity_id: input_boolean.heating_kitchen
    to: 'off'
condition: []
action:
  - choose:
      - conditions:
          - condition: state
            entity_id: input_boolean.heating_maintaince_kitchen
            state: 'on'
        sequence:
          - wait_for_trigger:
              - platform: state
                entity_id: input_boolean.heating_maintaince_kitchen
                from: 'on'
                to: 'off'
            timeout: '00:15:00'
          - service: climate.set_hvac_mode
            data:
              hvac_mode: 'off'
            entity_id: climate.radiator_valve_kitchen
      - conditions:
          - condition: state
            entity_id: input_boolean.heating_maintaince_kitchen
            state: 'off'
        sequence:
          - service: climate.set_hvac_mode
            data:
              hvac_mode: 'off'
            entity_id: climate.radiator_valve_kitchen
    default: []
mode: queued
max: 10
```
  
</details>

# 'heat rising too quickly'-detection

This automatism can detect sudden rises in temperature, like if someone is taking a shower in a room, does cooking or has an electrical device on, which generates a lot of heat. It can also detect a haywire reaction of the valve opening too long which would lead to an overshooting of the set temperature. This is important if you vented the room down several degrees and you're trying to heat it up again. This automatism will shut the valve to give the radiator more time to spread the heat in the room.

<details>
  
```yaml  
alias: 'Heating: Shut valve when the temperature is rising too fast - Kitchen'
description: and it's within 2 degrees of the set temperature
trigger:
  - platform: state
    entity_id: sensor.temp_sensor_kitchen_temperature
  - platform: state
    entity_id: sensor.radiator_valve_kitchen_position
  - platform: state
    entity_id: climate.radiator_valve_kitchen
    attribute: temperature
condition:
  - condition: numeric_state
    entity_id: sensor.derivation_filtered_temperature_kitchen
    above: '3'
  - condition: state
    entity_id: input_select.heating_mode_kitchen
    state: default
  - condition: state
    entity_id: input_boolean.heating_maintaince_kitchen
    state: 'off'
  - condition: template
    value_template: >-
      {{ (states("input_number.target_temp_kitchen") | float) <
      ((states("sensor.temp_sensor_kitchen_temperature") | float) + 2) }}
action:
  - service: input_select.select_option
    data:
      option: cut
    entity_id: input_select.heating_mode_kitchen
  - delay: '00:15:00'
mode: single
```

```yaml
alias: 'Heating: Turn the valve on again after 12 minutes cut - Kitchen'
description: ''
trigger:
  - platform: state
    entity_id: input_select.heating_mode_kitchen
    to: cut
    for: '00:12:00'
  - platform: time_pattern
    minutes: /15
condition:
  - condition: state
    entity_id: input_select.heating_mode_kitchen
    state: cut
    for: '00:11:55'
action:
  - choose:
      - conditions:
          - condition: not
            conditions:
              - condition: state
                entity_id: input_boolean.heating_maintaince_kitchen
                state: 'off'
        sequence:
          - wait_for_trigger:
              - platform: state
                entity_id: input_boolean.heating_maintaince_kitchen
                to: 'off'
            timeout: '00:11:55'
            continue_on_timeout: false
    default: []
  - service: input_select.select_option
    data:
      option: default
    entity_id: input_select.heating_mode_kitchen
  - delay: '00:03:00'
mode: single
```
                                                                         
</details>

# switch the heat mode for the window and heat sources

Those automatisms switch the input_select-heat modes according to the heat source and window sensors.

<details>
  
```yaml
alias: 'Heating: switch mode to off (window/heat source) - Kitchen'
description: ''
trigger:
  - type: opened
    platform: device
    entity_id: binary_sensor.window_sensor_kitchen_contact
    domain: binary_sensor
    for:
      seconds: 5
  - platform: state
    entity_id: input_boolean.heatsource_present_kitchen
    for:
      seconds: 5
    to: 'on'
    from: 'off'
  - platform: homeassistant
    event: start
condition:
  - condition: or
    conditions:
      - type: is_open
        condition: device
        entity_id: binary_sensor.window_sensor_kitchen_contact
        domain: binary_sensor
      - condition: state
        entity_id: input_boolean.heatsource_present_kitchen
        state: 'on'
action:
  - service: input_select.select_option
    data:
      option: 'off'
    entity_id: input_select.heating_mode_kitchen
mode: queued
max: 10
```
  

```yaml
alias: 'Heating: switch mode to on (window/heat source) - Kitchen'
description: ''
trigger:
  - type: not_opened
    platform: device
    entity_id: binary_sensor.window_sensor_kitchen_contact
    domain: binary_sensor
    for:
      seconds: 5
  - platform: state
    entity_id: input_boolean.heatsource_present_kitchen
    from: 'on'
    to: 'off'
    for: '00:15:00'
  - platform: homeassistant
    event: start
condition:
  - type: is_not_open
    condition: device
    entity_id: binary_sensor.window_sensor_kitchen_contact
    domain: binary_sensor
  - condition: state
    entity_id: input_boolean.heatsource_present_kitchen
    state: 'off'
action:
  - service: input_select.select_option
    data:
      option: default
    entity_id: input_select.heating_mode_kitchen
mode: queued
max: 10
```

</details>

# thermostat controller

The main temperature controller automatism. Gets triggered by some events pretty regularily, but the delay at the end and the single execution makes sure it only runs once a minute at most - to avoid too many valve operations.

The **first condition** in the `choose` makes sure that if the temperature is within -0.5°C the heating is reduced properly by the valve. If not the valve is closed and the valve's automtion restarted at 0%, which ususlly let it open again slowly - if neccessary - on it's own.

When it's 30% or below it's ususally already detecting the approached set temperature right, and will close soon.

The **second condition** in the `choose` just updates the set temperature on the valve (relative to the measured temperature of the valve itself) as long as the set temperature isn't reached.

<details>
  
```yaml
alias: 'Heating: Set temperature for default mode - Kitchen v4.2'
description: ''
trigger:
  - platform: state
    entity_id: input_number.target_temp_kitchen
  - platform: state
    entity_id: input_boolean.heating_kitchen
    to: 'on'
  - platform: state
    entity_id: input_select.heating_forced_off_weather
    to: 'no'
  - platform: state
    entity_id: input_select.heating_mode_kitchen
    to: default
  - platform: state
    entity_id: sensor.temp_sensor_kitchen_temperature
    for: '00:00:15'
  - platform: state
    entity_id: input_boolean.heating_maintaince_kitchen
    from: 'on'
    to: 'off'
    for: '00:01:00'
  - platform: numeric_state
    entity_id: sensor.radiator_valve_kitchen_position
    above: '40'
    for: '00:02:00'
  - platform: numeric_state
    entity_id: sensor.radiator_valve_kitchen_position
    above: '60'
    for: '00:02:00'
  - platform: numeric_state
    entity_id: sensor.radiator_valve_kitchen_position
    for: '00:02:00'
    above: '80'
  - platform: numeric_state
    entity_id: sensor.radiator_valve_kitchen_position
    for: '00:02:00'
    above: '90'
  - platform: numeric_state
    entity_id: sensor.radiator_valve_kitchen_position
    for: '00:02:00'
    above: '99'
  - platform: state
    entity_id: input_number.target_temp_kitchen
    for: '00:01:00'
condition:
  - condition: state
    entity_id: input_select.heating_mode_kitchen
    state: default
  - condition: state
    entity_id: input_select.heating_forced_off_weather
    state: 'no'
  - condition: state
    entity_id: input_boolean.heating_kitchen
    state: 'on'
  - condition: state
    entity_id: input_boolean.heating_maintaince_kitchen
    state: 'off'
action:
  - choose:
      - conditions:
          - condition: template
            value_template: >-
              {{ ((states("input_number.target_temp_kitchen") | float) - 0.5) <
              (states("sensor.temp_sensor_kitchen_temperature") | float) }}
        sequence:
          - choose:
              - conditions:
                  - condition: numeric_state
                    entity_id: sensor.radiator_valve_kitchen_position
                    above: '30'
                sequence:
                  - service: climate.set_hvac_mode
                    data:
                      hvac_mode: 'off'
                    entity_id: climate.radiator_valve_kitchen
                  - wait_for_trigger:
                      - platform: numeric_state
                        entity_id: sensor.radiator_valve_kitchen_position
                        below: '1'
                    timeout: '00:01:00'
          - service: climate.set_hvac_mode
            data:
              hvac_mode: auto
            entity_id: climate.radiator_valve_kitchen
          - delay: '00:00:10'
          - service: climate.set_temperature
            data_template:
              entity_id: climate.radiator_valve_kitchen
              temperature: >-
                {{ ((states("input_number.target_temp_kitchen") | float -
                states("sensor.temp_sensor_kitchen_temperature") | float ) +
                state_attr("climate.radiator_valve_kitchen",
                "local_temperature") | float) | round(1, "ceil")}}
      - conditions:
          - condition: template
            value_template: >-
              {{ (states("input_number.target_temp_kitchen") | float) >
              (states("sensor.temp_sensor_kitchen_temperature") | float) }}
        sequence:
          - service: climate.set_hvac_mode
            data:
              hvac_mode: auto
            entity_id: climate.radiator_valve_kitchen
          - delay: '00:00:10'
          - service: climate.set_temperature
            data_template:
              entity_id: climate.radiator_valve_kitchen
              temperature: >-
                {{ ((states("input_number.target_temp_kitchen") | float -
                states("sensor.temp_sensor_kitchen_temperature") | float ) +
                state_attr("climate.radiator_valve_kitchen",
                "local_temperature") | float) | round(1, "ceil")}}
    default: []
  - delay: '00:01:00'
mode: single
```

</details>

# a simple weather detection

This detection tries to detect two different type of weather scanarios:

- Hot days, where you like to get some cool air into your house to chill it down in the night. You don't want no heating to turn on on those days, even in the night.
- Mixed bag weather, like when it's around 20°C and bit cooler in the night, but you don't want your central heating system to turn on to inrease the temperature in all rooms by say 1°C.

<details>
  
```yaml  
alias: 'Heating: Weather detection '
description: ''
trigger:
  - platform: time_pattern
    minutes: '23'
    seconds: '42'
condition: []
action:
  - choose:
      - conditions:
          - condition: numeric_state
            entity_id: sensor.temperature_statistics_24h
            attribute: max_value
            above: '22'
        sequence:
          - service: input_select.select_option
            data:
              option: hot-day
            entity_id: input_select.heating_forced_off_weather
      - conditions:
          - condition: numeric_state
            entity_id: sensor.temperature_statistics_24h
            attribute: min_value
            above: '15'
        sequence:
          - service: input_select.select_option
            data:
              option: no-heating-required
            entity_id: input_select.heating_forced_off_weather
    default:
      - service: input_select.select_option
        data:
          option: 'no'
        entity_id: input_select.heating_forced_off_weather
mode: single
```  
  
</details>

# home assistant startup

On startup all valves will be closed and the heating mode will be set to the internal value `default` to make sure the automatism has a defined starting point.

<details>

```yaml
alias: 'Heating: close valves and deactivate maintaince mode when starting'
description: ''
trigger:
  - platform: homeassistant
    event: start
condition: []
action:
  - service: climate.set_hvac_mode
    data:
      hvac_mode: 'off'
    entity_id: climate.radiator_valve_kitchen
  - service: input_boolean.turn_off
    data: {}
    entity_id: input_boolean.heating_maintaince_kitchen
  - service: input_select.select_option
    data:
      option: default
    entity_id: input_select.heating_mode_kitchen
mode: restart
```

</details>

# valve maintaince

In summer the valves usually get never moved, this automation moves the valves once a day to avoid that the valves get sticky or stuck.

<details>

```yaml
alias: 'Heating: Clean valves once per day (when heating mode is default)'
description: ''
trigger:
  - platform: time
    at: '04:47:33'
condition: []
action:
  - service: input_boolean.turn_on
    data: {}
    entity_id: input_boolean.heating_maintaince_kitchen
  - service: climate.set_hvac_mode
    data:
      hvac_mode: 'off'
    entity_id: climate.radiator_valve_kitchen
  - delay: '00:01:00'
  - service: climate.set_hvac_mode
    data:
      hvac_mode: heat
    entity_id: climate.radiator_valve_kitchen
  - delay: '00:01:00'
  - service: climate.set_hvac_mode
    data:
      hvac_mode: 'off'
    entity_id: climate.radiator_valve_kitchen
  - delay: '00:01:00'
  - service: climate.set_hvac_mode
    data:
      hvac_mode: heat
    entity_id: climate.radiator_valve_kitchen
  - delay: '00:01:00'
  - service: climate.set_hvac_mode
    data:
      hvac_mode: 'off'
    entity_id: climate.radiator_valve_kitchen
  - service: input_boolean.turn_off
    data: {}
    entity_id: input_boolean.heating_maintaince_kitchen
mode: restart
```

</details>
