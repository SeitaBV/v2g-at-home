# FlexMeasures / Home Assistant / Wallbox Quasar integration

FlexMeasures integration for a Wallbox Quasar connected to Home Assistant using AppDaemon.

This integration lets you add smart charge control to your Wallbox Quasar. Optimize the upcoming hours with an eye on energy prices, your solar generation or the CO2 content of the grid. (For now: only energy prices, the rest is to come)

In practice, you can do the following via your Home Assistant, you can 

- Switch the charging strategy between user and automatic
- In automatic mode, FlexMeasures is periodically asked to generate schedules, which Home Assistant translates into set points which it sends to the Wallbox Quasar via modbus.
- Set targets (e.g. be charged 100% at 7am tomorrow) which prompts FlexMeasures to update its schedules.


# Installation

The installation tutorial assumes you have already installed Home Assistant, including the AppDaemon 4 add-on.

In Home Assistant, look for the AppDaemon configuration (`Supervisor -> AppDaemon 4 -> Configuration`) and add the following Python packages:

```yaml
python_packages:
  - isodate
  - pyModbusTCP
```

In your Home Assistant file editor, go to `/config/appdaemon/apps/` and add `fm_ha_integration.py`.

In the same directory, add (or extend) `apps.yaml` with (replacing secrets as required for your custom setting):

```yaml
---
flexmeasures-home-assistant:
  module: fm_ha_integration
  class: FlexMeasuresWallboxQuasar
  fm_api: https://seita.flexmeasures.io/api
  fm_api_version: v2_0
  fm_user_email: !secret fm_user_email
  fm_user_password: !secret fm_user_password
  fm_quasar_entity_address: !secret fm_quasar_entity_address
  reschedule_on_soc_changes_only: true
  max_number_of_reattempts_to_retrieve_device_message: 2
  delay_for_reattempts_to_retrieve_device_message: 60
  delay_for_initial_attempts_to_retrieve_device_message: 5
  wallbox_host: !secret wallbox_host
  wallbox_port: !secret wallbox_port
  wallbox_register_get_state_of_charge: !secret wallbox_register_get_state_of_charge
  wallbox_register_set_power_setpoint: !secret wallbox_register_set_power_setpoint
  wallbox_register_set_control: !secret wallbox_register_set_control
  wallbox_register_set_setpoint_type: !secret wallbox_register_set_setpoint_type
  wallbox_register_set_setpoint_type_value_current: !secret wallbox_register_set_setpoint_type_value_current
  wallbox_register_set_setpoint_type_value_power_by_phase: !secret wallbox_register_set_setpoint_type_value_power_by_phase
  wallbox_register_set_control_value_user: !secret wallbox_register_set_control_value_user
  wallbox_register_set_control_value_remote: !secret wallbox_register_set_control_value_remote
```

In `/config/configuration.yaml` add the following Modbus sensor to get a signal of your car's state of charge, and some input fields to store a clean SoC signal, a selected charging strategy and charging schedules:

```yaml

modbus:
  - name: quasar
    delay: 5
    timeout: 4
    type: tcp
    host: !secret wallbox_host
    port: !secret wallbox_port
    sensors:
      - name: charger_connected_car_state_of_charge
        address: !secret wallbox_register_get_state_of_charge
        input_type: holding
        data_type: int16
        scan_interval: 120
        unit_of_measurement: "%"
        slave: 1
input_number:
  car_state_of_charge:
    name: Car State of Charge
    icon: mdi:battery-medium
    min: 0
    max: 100
    step: 1
    unit_of_measurement: %
input_select:
  charging_strategy:
    name: Charging strategy
    icon: mdi:battery-charging-medium
    options:
      - Automatic
      - Forced ON
      - Forced OFF
input_text:
  chargeschedule:
    name: ChargeSchedule
    icon: mdi:calendar-multiselect
```

In `/config/automations.yaml` add:

```yaml
- id: '1626364003549'
  alias: Clean up SoC change as reported by Quasar
  description: "Goal: make the technical state of charge (from Modbus YAML in configuration)\
    \ better readable.\nThe Quasar returns an SoC of 0% if the charger is not connected\
    \ /paused. This is not the true state of charge.\nThis is why we fill another (input)\
    \ variable with the most recent non-zero value.\nThe Modbus YAML code also frequently\
    \ returns \"unavailable\". We also ignore that here.\nTo better estimate the true state\
    \ of charge, we also store the datetime of a (correct) change, as another input number."
  trigger:
  - platform: state
    entity_id: sensor.charger_connected_car_state_of_charge
  condition:
  - condition: numeric_state
    entity_id: sensor.charger_connected_car_state_of_charge
    above: '1'
  - condition: not
    conditions:
    - condition: state
      entity_id: sensor.charger_connected_car_state_of_charge
      state: unavailable
  action:
  - service: input_number.set_value
    target:
      entity_id: input_number.car_state_of_charge
    data:
      value: '{{states(''sensor.charger_connected_car_state_of_charge'')}}'
  mode: single
```
