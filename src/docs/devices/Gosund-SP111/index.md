---
title: Gosund SP111
date-published: 2021-01-01
type: plug
standard: eu
board: esp8266
---

## Warning

![Warning](https://upload.wikimedia.org/wikipedia/commons/thumb/1/17/Warning.svg/260px-Warning.svg.png)

Looks like device chip was replaced to non-flashable custom tasmota chip.

Please be aware, that there is a new version of that outlet, often having the phrase `EP2` instead or in addition to
`SP111`, sold starting in November 2020. For that version, the tuya script does not longer work! Also a breakless
opening of the plug is much harder due to a removed screw on the bottom of the device.

## Flashing

The older devices can be flashed [using tuya-convert](/devices/tuya-convert). Fresh out of the factory it will be in
autoconfig mode. When plugged in for the first time tuya-convert will pick it up directly.

![Hardly visible screw on original SP111](./gosund-sp111.JPG "Hardly visible screw on original SP111")

Make sure the plug has that screw on the bottom!

## GPIO Pinout

[see pinout](https://templates.blakadder.com/gosund_SP111_v1_1.html)

| Pin    | Function   |
| ------ | ---------- |
| GPIO00 | Led1i      |
| GPIO01 | None       |
| GPIO02 | LedLinki   |
| GPIO03 | None       |
| GPIO04 | HLWBL CF1  |
| GPIO05 | BL0937 CF  |
| GPIO09 | None       |
| GPIO10 | None       |
| GPIO12 | HLWBL SELi |
| GPIO13 | Button1    |
| GPIO14 | None       |
| GPIO15 | Relay1     |
| GPIO16 | None       |

## Basic Configuration

```yaml
substitutions:
  devicename: "Gniazdo_Salon"
  upper_devicename: "Gosund SP111"
  current_res: "0.00120"
  voltage_div: "732"

esphome:
  name: gniazdkosalon
  friendly_name: Gniazdo Salon

esp8266:
  board: esp01_1m
  board_flash_mode: dout

wifi:
  networks:
    - ssid: !secret wifi_ssid
      password: !secret wifi_password
    - ssid: !secret wifi_ssid2
      password: !secret wifi_password2

  ap:
    ssid: "gniazdosalon"
    password: !secret wifi_ap_password

  min_auth_mode: WPA2
  manual_ip:
    static_ip: 10.0.0.56
    gateway: 10.0.0.1
    subnet: 255.255.255.0

logger:
  level: INFO

web_server:
  port: 80
  version: 3
  auth:
    username: !secret web_server_username
    password: !secret web_server_password

time:
  - platform: sntp
    id: sntp_time
    timezone: Europe/Warsaw
    servers:
      - ntp1.tp.pl
      - 1.pool.ntp.org
      - 2.pool.ntp.org
  - platform: homeassistant
    id: homeassistant_time

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

captive_portal:

####################################
# TEXT SENSOR
####################################
text_sensor:
  - platform: version
    name: "${devicename} - Wersja"
    icon: mdi:cube-outline

####################################
# BINARY SENSOR
####################################
binary_sensor:
  - platform: status
    name: "${devicename} - Status"
    device_class: connectivity
    icon: mdi:lan-connect

  - platform: gpio
    pin:
      number: GPIO13
      mode: INPUT_PULLUP
      inverted: true
    id: "${devicename}_button_state"
    icon: mdi:gesture-tap-button
    on_press:
      - switch.toggle: button_switch

####################################
# SENSORS
####################################
sensor:
  - platform: wifi_signal
    name: "${devicename} - WiFi"
    update_interval: 60s
    icon: mdi:wifi

  - platform: uptime
    name: "${devicename} - Uptime"
    update_interval: 60s
    icon: mdi:clock-outline

  - platform: total_daily_energy
    name: "${devicename} - Dzienne Zużycie"
    power_id: "power_wattage"
    filters:
      - multiply: 0.001
    unit_of_measurement: kWh
    icon: mdi:calendar-clock

  - platform: adc
    pin: VCC
    name: "${devicename} - VCC"
    icon: mdi:flash-outline

  - platform: hlw8012
    sel_pin:
      number: GPIO12
      inverted: true
    cf_pin: GPIO05
    cf1_pin: GPIO04
    change_mode_every: 4
    current_resistor: ${current_res}
    voltage_divider: ${voltage_div}
    update_interval: 3s

    current:
      id: current_sensor
      name: "${devicename} - Prąd"
      unit_of_measurement: A
      accuracy_decimals: 3
      icon: mdi:current-ac

    voltage:
      id: voltage_sensor
      name: "${devicename} - Napięcie"
      unit_of_measurement: V
      accuracy_decimals: 1
      icon: mdi:flash-outline

    power:
      id: power_wattage
      name: "${devicename} - Moc"
      unit_of_measurement: W
      accuracy_decimals: 1
      icon: mdi:gauge

####################################
# Power Factor / Moc pozorna / Moc bierna
####################################
  - platform: template
    name: "${devicename} - Power Factor"
    unit_of_measurement: ""
    accuracy_decimals: 2
    icon: mdi:angle-acute
    lambda: |-
      if (id(voltage_sensor).has_state() && id(current_sensor).has_state()) {
        float pf = id(power_wattage).state / (id(voltage_sensor).state * id(current_sensor).state);
        if (pf > 1.0) return 1.0;
        return pf;
      }
      return 0.0;

  - platform: template
    name: "${devicename} - Moc pozorna"
    unit_of_measurement: VA
    accuracy_decimals: 1
    icon: mdi:sine-wave
    lambda: |-
      return id(voltage_sensor).state * id(current_sensor).state;

  - platform: template
    name: "${devicename} - Moc bierna"
    unit_of_measurement: var
    accuracy_decimals: 1
    icon: mdi:triangle-outline
    lambda: |-
      float s = id(voltage_sensor).state * id(current_sensor).state;
      float p = id(power_wattage).state;
      if (s > p) return sqrt((s * s) - (p * p));
      return 0.0;

####################################
# LED
####################################
status_led:
  pin:
    number: GPIO02
    inverted: true
  id: led_blue

output:
  - platform: gpio
    pin: GPIO00
    inverted: true
    id: led_red

####################################
# SWITCH
####################################
switch:
  - platform: template
    name: "${devicename} - Switch"
    icon: mdi:power-socket-eu
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON
    lambda: "return id(relay).state;"
    id: button_switch
    turn_on_action:
      - switch.turn_on: relay
      - output.turn_on: led_red
    turn_off_action:
      - switch.turn_off: relay
      - output.turn_off: led_red

  - platform: gpio
    pin: GPIO15
    id: relay
```
