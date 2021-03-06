## Problem
My home is 4 stories and the builder was smart to put the furnace on the 4th floor. This made it super hard for the furnace to push heat down to the first floor bedroom that I use as an office. After working with them, they decided to install a 240V heater in the 1st floor bedroom and use a line voltage thermostat to controll it. This solution is not ideal since:
1. It is an electrical solution, now I have to pay more boo :(
2. It is controlled using old school thermostats that make it harder to control remotely. In the mornings it will take about 1 hour to heat up the room and I sometimes forget to turn it off (more money wasted boo)

## Making it a Smart Heater
The thermostat is a single-pole line voltage thermostat. Since the thermostat is manual, it is using a metal plate that expand based on the temperature and then if its too cold it would release a button that is pushed by default to turn it on. so when button is pushed the switch is closed and when its not pressed the switch is open and the power flows. This is important since it means if the button is not pressed the heater is on and if the button is pressed the the heater is off. The circuit diagram looks like this:
<p>
    <img src="images/heater/circuit.png" width="400" />
</p>

The physical button is the white small button under the lever, can see it here
<p>
    <img src="images/heater/thermometer_button.jpeg" width="400" />
</p>

### Minimally Invasive Solution
I tried making it smart. First, I tried to get a button pusher thats zwave controlled (see pic below) to just push or release the button. but it was super hard to calibrate it cuz its not powerful enough for the button. The button is very hard to push so it tries to push it and then backs off. So I tried to make a long lever to increase the torque, but the lever was a little flimsy and hard to keep from moving around. also the fact that when the button isn't pressed it means power flows and heater is on made it scary. If the button or lever fails then the heater will be on until I physically go and fix it. So not a great solution.
<p>
    <img src="images/heater/switchbot_button_pusher.jpg" width="400" />
</p>

### More Invasive Solution
I ended up switching over to a sonoff switch. so basically took out the thermostat and put in a sonoff switch. its working really nicely now! can control from HomeAssistant and configured it to be a thermostat using hass by using the Bluetooth thermometer as input and the sonoff switch as the heater! so now its just as if its a real thermostat where it reads input from this BLE thermostat and then turning on the heater when needed. The awesome part is that homeassistant already has a default climate component where you provide the temp sensor and the heater entity and then it lets you set your desired temperature and it will do all the work! This also gets exposed to alexa so I can change the temperature by talking to Alexa. see below for pics. 

This the old thermostat:
<p>
    <img src="images/heater/thermometer_front.jpeg" width="400" />
</p>

This is the sonoff switch:
<p>
    <img src="images/heater/Sonoff_installed.jpeg" width="400" />
</p>

This is the Bluetooth thermometer:
<p>
    <img src="images/heater/BLE_Thermometer.jpeg" width="400" />
</p>

Homeassistant climate UI component:
<p>
    <img src="images/heater/HA_Climate_Control.jpeg" width="400" />
</p>

The completed product once everything is hidden, calling it the Frankenstat:
<p>
    <img src="images/heater/frankenstat.jpg" width="400"/>
</p>

#### Sonoff Component Setup
It is recommended to flash tasmota onto sonoff switches, but the sonoff switch I'm using is the Sonoff Basic Switch R2. The R2 is harder to flash since it cannot be flashed [OTA without some tweaking](https://tasmota.github.io/docs/devices/Sonoff-Basic/#sonoff-basic-r2). Instead, I decided to use [SonoffLan](https://github.com/AlexxIT/SonoffLAN) custom component to control sonoff switches. To set this up, I added this custom repo into HACS and then installed 'Sonoff Lan' component. Once all installed, I configured switch in the configuartion file like so:

```
sonoff:
  username: [username]
  password: [password]
```

#### HA Configuration
To setup the configuration just need to use the ['Generic Thermostat'](https://www.home-assistant.io/integrations/generic_thermostat) integration and point the heater to the sonoff switch and the target_sensor to the BLE thermometer. For me this was:

```
climate:
  - platform: generic_thermostat
    name: Office
    heater: switch.office_heater
    target_sensor: sensor.a4_c1_38_4d_d9_d1_temp
    precision: 0.1
```
