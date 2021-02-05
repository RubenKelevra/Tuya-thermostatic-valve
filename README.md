# Tuya-thermostatic-valve
automation documentation for better temperture control, with an external temperature probe / window contact


# prerequisites

- Home Assistant (2021.2.0 at the time of writing)
- Properly setup Zigbee2MQTT plugin (or similar)
- properly installed
  - [TuYa TS0601](https://www.zigbee2mqtt.io/devices/TS0601_thermostat.html) thermostat
  - [Aqara window contact sensor](https://www.zigbee2mqtt.io/devices/MCCGQ11LM.html) (or similar)
  - [TuYa TS0121 plug](https://www.zigbee2mqtt.io/devices/TS0121_plug.html) (or similar) for any heat source
- latest thermostat firmware
- thermostat settings:
  - Make sure the build-in "window detection" is off
  - Set minimal temperature to 1°C
  - Set maximum temperature to something high, like 35°C
  - Set hysteresis to 0.5°C (default is 1.0°C)
  - Make sure the valve control type is set to "PID"

# recommended mods

- Attach a 3 V power supply at the pcb to avoid fast battery drainage due to larger/more often valve operation, see [install-powersupply](power_supply_mod.md)
