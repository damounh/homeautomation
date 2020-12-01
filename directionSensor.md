## Purpose/Problem
My home has 4 stories and each floor is open concept. This makes it annoying to have to keep turning on lights and easy to forget to turn off lights. Instead of having to ask Alexa or use the HomeAssistant app to turn the lights on or off, it would be great if I can detect the number of occupants on each floor and turn the lights on and off. This can easily be done using PIR motion sensors to turn lights on and off, but I wanted to see if I can also count the number of people on each floor. By being able to know the number of people in the house and the number of people on each floor, it would open up opportunity for more automations.

## Possible Solutions
There are multiple ways of counting the number of people in a room/area/floor.

### Room-Assistant
One way I keep coming across is to use [room-assistant](https://www.room-assistant.io/) which can use a number of sensors to determine occupancy of a room. Some of the sensors/integrations that can be used in room-assistant are:
* Bluetooth Low Energy, which scans for devices that advertise, such as iBeacon or Bluetooth tags
* Bluetooth Classic, which tries to detect the location of Bluetooth devices within the home
* Thermopile sensors that measure the temperature on an 8x8 grid

#### Pros
* Well supported project with a strong community
* Advice and gotchas related to placement of sensors
* Number of supported integrations that can help improve accuracy
* Configuration for integrations like BLE signal strength are supported out of the box

#### Cons
* I'm not a big fan of the first two bluetooh sensors since I don't want to have to carry my phone or a BLE device with me everywhere in the house.
* Thought dump (likely resolved): The thermopile solution would work and will be pretty cool way of doing it, but the logic to count the number of people on each floor might get a little complicated. Either I would have to have multiple per floor so I can constantly monitor all people on each floor or I would have to write logic for incrementing and decrementing occupancies as people appear and disappear on each floor. The logic would get a bit complicated as I will not get direction out of the box with thermopile.

### Direction Sensor using ToF
Since my house is 4 floors, and all I want is to determine the number of people on each floor, I could build 3 'direction sensors' by putting two ToF sensors and an ESP8266 or ESP32. The arrangement of the two ToF sensors will can then be used to determine the direction of movement. I can then place each 'direction sensor' at each stair case to see if someone is going up or down the stairs. The circuit can be seen below (please excuse the constant hopping, this is a prototype)

<p>
    <img src="images/direction_sensor/direction_sensor_circuit_full.jpg" width="500" />
</p>

#### Pros
* No hardware required to be carried by people walking around (ie. no need to carry iBeacons or phones)
* It's fun to pretend I know what I'm doing

#### Cons
* Cannot identify the person that is passing by
* Adding other occupancy sensors, like PIR sensors, to determine presense will have to be done manually
* Will have to manually tune the ESP code to consume less power

## Proposal
Since I don't always carry my phone with me and would not like to carry around an iBeacon, I dont think I would take advantage of room-assistant's BLE or Bluetooth Classic integrations. Also, unlike the logic required for thermopile, it should be less effort to go with the ToF solution since I would know the direction of movement at each staircase and won't have to make assumptions. Because of these reasons, I think the ToF approach would be a good start.

## Hardware Requirements
* Home Assistant Configured
* ESPHome
* ESP8266 or ESP32
* 2x VL53L0X

### ESP Logic
oh boy this logic needs to be explained, it had a lot of iterations, seems to be the one that works best for now

```
esphome:
  name: tof_first_floor
  platform: ESP32
  board: esp-wrover-kit

globals:
 - id: sensor_1_is_active
   type: bool
   restore_value: no
   initial_value: 'false'
 - id: sensor_2_is_active
   type: bool
   restore_value: no
   initial_value: 'false'

i2c:
  - sda: 18
    scl: 5
    scan: True
    id: bus_a
  - id: bus_b
    sda: 23
    scl: 22
    scan: True

text_sensor:
  - platform: template
    name: "First Floor Stair Case Direction"
    id: first_floor_stair_case_direction

sensor:
  - platform: vl53l0x
    name: "First Floor Sensor 1 Distance"
    address: 0x29
    update_interval: 47ms
    long_range: False
    internal: True
    i2c_id: bus_a
    id: sensor_1
    on_value:
      then:
        if:
          condition:
              lambda: |-
                return id(sensor_1).state < 0.8 && id(sensor_1).state > 0 && !id(sensor_1_is_active);
          then:
            - globals.set:
                id: 'sensor_1_is_active'
                value: 'true'
            - logger.log:
                format: "SENSOR 1 ACTIVE %f, SENSOR 2 ACTIVITY IS %s"
                args: [ 'id(sensor_1).state', 'id(sensor_2_is_active) ? "true" : "false"' ]
                level: INFO
            - if:
                condition:
                  lambda: |-
                    return id(sensor_2_is_active);
                then:
                  - text_sensor.template.publish:
                      id: first_floor_stair_case_direction
                      state: "Down"
                  - logger.log:
                      format: "Publishing DOWN"
                      level: INFO
                  - delay: 1s
                  - text_sensor.template.publish:
                      id: first_floor_stair_case_direction
                      state: "None"
                else:
                  - logger.log:
                      format: "Did not publish down, SENSOR 1 ACTIVITY IS %s and SENSOR 2 ACTIVITY IS %s"
                      args: [ 'id(sensor_1_is_active) ? "true" : "false"', 'id(sensor_2_is_active) ? "true" : "false"' ]
                      level: INFO
            - delay: 2s
            - globals.set:
                id: 'sensor_1_is_active'
                value: 'false'
  - platform: vl53l0x
    name: "First Floor Sensor 2 Distance"
    address: 0x29
    update_interval: 43ms
    long_range: False
    internal: True
    i2c_id: bus_b
    id: sensor_2
    on_value:
      then:
        if:
          condition:
              lambda: |-
                return id(sensor_2).state < 0.8 && id(sensor_2).state > 0 && !id(sensor_2_is_active);
          then:
            - globals.set:
                id: 'sensor_2_is_active'
                value: 'true'
            - logger.log:
                format: "SENSOR 2 ACTIVE %f, SENSOR 1 ACTIVITY IS %s"
                args: [ 'id(sensor_2).state', 'id(sensor_1_is_active) ? "true" : "false"' ]
                level: INFO
            - if:
                condition:
                  lambda: |-
                    return id(sensor_1_is_active);
                then:
                  - text_sensor.template.publish:
                      id: first_floor_stair_case_direction
                      state: "Up"
                  - logger.log:
                      format: "Publishing UP"
                      level: INFO
                  - delay: 1s
                  - text_sensor.template.publish:
                      id: first_floor_stair_case_direction
                      state: "None"
                else:
                  - logger.log:
                      format: "Did not publish up, SENSOR 1 ACTIVITY IS %s and SENSOR 2 ACTIVITY IS %s"
                      args: [ 'id(sensor_1_is_active) ? "true" : "false"', 'id(sensor_2_is_active) ? "true" : "false"' ]
                      level: INFO
            - delay: 2s
            - globals.set:
                id: 'sensor_2_is_active'
                value: 'false'

wifi:
  ssid: "{SSID}"
  password: "{SSID_PASSWORD}"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Tof First Floor Fallback Hotspot"
    password: "{AP_PASSWORD}"

captive_portal:

# Enable logging
logger:
  level: INFO

# Enable Home Assistant API
api:

ota:

```

### Home Assistant Configuration
The only home assistant configuration was to add the ESPs that have been deployed through ESPHome to HA. This was done by through integrations:

<p>
    <img src="images/direction_sensor/esp_home_ha_integrations.png" width="300" />
</p>

#### Automation Configuration
Created two counter in automation Helpers to track occupancy for Main Floor (2nd floor) and 3rd Floor:

<p>
    <img src="images/direction_sensor/automation_helpers_counters.png" width="750" />
</p>

<p>
    <img src="images/direction_sensor/main_floor_occupant_counter.png" width="300" />
</p>

Automation to increment people in living room:
```
alias: 'Presence: Going to Living Room from 1st Floor'
description: ''
trigger:
  - platform: state
    entity_id: sensor.first_floor_stair_case_direction
    to: Up
condition: []
action:
  - service: counter.increment
    data: {}
    entity_id: counter.occupants_main_floor
mode: single
```

Automation to turn lights on if anyone is in living room:
```
alias: 'Presence: No one in living room'
description: ''
trigger:
  - platform: state
    entity_id: counter.occupants_main_floor
    to: '0'
condition: []
action:
  - service: light.turn_off
    data: {}
    entity_id: light.kitchen_ceiling
  - service: light.turn_off
    data: {}
    entity_id: light.island
  - service: light.turn_off
    data: {}
    entity_id: light.living_room
mode: single

```

## Result
It took a lot of trial an error to tune the code. Initially, I was having the ESP report the distance each ToF sensor was reading and was performing the direction determination logic within the HomeAssistant automation. Unfortunately, this proved to result to many false positive. {Need to expand on this part}. Instead, decided to move the logic onto the ESP and have the ESP determine the direction. This proved to be much more reliable.

For now, there are far less false positives. {Need to add statistics here}