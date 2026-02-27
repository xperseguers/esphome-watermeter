This is my solution to monitor my home's water usage at very low cost by reading my analog water meter:

<p align="center">
  <img src="images/water-meter.jpg" width="250"> 
</p>

There are other projects that exist to measure your water meter. ESP32 has even the capability to run
[machine-vision software](https://github.com/jomjol/AI-on-the-edge-device) to decode the dials on your
meter, and parse them into data that Home Assistant can understand, but it was way too complex to start
with and trying to use a cheap proximity sensor (inductive sensor) proved to be the perfect match:

<p align="center">
  <img src="images/proximity-sensor.jpg" width="250"> 
</p>


## Context

I own a multi-jet turbine meter for cold water
[aquabasic® PMK-basic](https://ch.integra-metering.com/fr/product/aquabasic-pmk-basic/) which is
widely used in Switzerland. 

The turbine meter is featuring a rotary meter in cubic meter, and 4 analog dials showing hundreds of
liters (×0.1), tens of liters (×0.01), liters (×0.001), and deciliters (×0.0001). That last dial is
the one that is featuring a semi-circular metal disc that we aim at detecting thanks to the inductive
sensor. 1 rotation cycle = 1 liter, thus measuring each liter consumption.

<p align="center">
  <img src="images/3d-mount.jpg" width="250">
  <img src="images/3d-encloser.jpg" width="250">
</p>


## Hardware

- 1× Proximity sensor [NPN NO LJ18A3-8-Z/BX-5V](https://de.aliexpress.com/item/1005004867517992.html)
- 1× WeMos D1 Mini TYPE-C ESP8266 ESP-12F
- 1× Mini prototype board from [uPesy](https://www.upesy.com/products/upesy-protoboard-breadboard-mini)
- 3× Threaded inserts M3 (2×5mm height for the board, 1×4mm height for the cover)
- 3× Cross round Phillips pan head screw bolt M3, length 6mm
- Straight pin PCB screw terminal block connector (I used 2× KF301-2P, but you could easily get a 1× KF301-3P)

You can 3D-print the mount and encloser with [aquabasic.stl](aquabasic.stl).


## Code

```yaml
esphome:
  name: water-meter
  friendly_name: water-meter
  on_boot:
    then:
      - pulse_meter.set_total_pulses: 
          id: water_pulse_meter
          value: !lambda 'return id(water_meter_total);'

esp8266:
  board: esp01_1m
  restore_from_flash: true

preferences:
  flash_write_interval: 5min

globals:
  id: water_meter_total
  type: int
  restore_value: yes

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "--redacted--"
  actions:
    - action: set_total_liters
      variables:
        new_total_liters: int
      then:
        - pulse_meter.set_total_pulses:
            id: water_pulse_meter
            value: !lambda 'return new_total_liters;'

ota:
  - platform: esphome
    password: "--redacted--"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  min_auth_mode: WPA

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Water-Meter Fallback Hotspot"
    password: "--redacted--"

captive_portal:

sensor:
  - platform: pulse_meter
    pin: GPIO12
    name: "Water Meter Flow"
    id: water_pulse_meter
    unit_of_measurement: "liter/min"
    icon: "mdi:water"
    total:
      name: "Water Meter Total"
      unit_of_measurement: "m³"
      accuracy_decimals: 3
      device_class: water
      state_class: total_increasing
      filters:
        - lambda: |-
            id(water_meter_total) = id(water_pulse_meter).raw_state;
            return x / 1000.0;
```

**Note:** In order to preset the total number of your counter, you can open
Home Assistant > Parameters > Developer Tools > Actions and make use of the API action
defined in the YAML definition.