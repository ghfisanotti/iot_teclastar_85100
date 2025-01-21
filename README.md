# Flashing ESPhome to a Tuya/Beken WB2S powered WiFi smart switch

This is a WiFi Switch powered by a Tuya/Beken WB2S module with a BK7231T MCU, it is sold in my country with the brand TeclaStar, Model 85100

It's used with the Tuya app installed in Android.

## Hacking it to be used locally with HomeAssistant

Although it is possible to somewhat integrate Tuya devices in the cloud with HomeAssistant, 
I prefer a local integration, in order to do that, I changed the firmware of the WB2S for
ESPhome, which also required to reverse engineer the connections of the relay, the button
and the LED.

After some fiddling with the multi-tester and some trying, I determined these are the
connections of the WB2S modules:

~~~
button -> pin PWM5 (P26)
relay  -> pin PWM4 (P24)
led    -> pin PWM1 (P7)
~~~

## Generating ESPhome firmware

This can be done with the ESPhome Dashboard, one way is running this in Docker:

~~~
docker run --rm --detach --name esphome \
  -v /home/gfisanot/esphome:/config \
  --net=host esphome/esphome
~~~

Then you can access the dashboard using Google Chrome at http://localhost:6052

Create a new BK72XX device and select WB2S as the board. Then replace the default
template with the following YAML code:

~~~
substitutions:
  device_name: teclastar1

esphome:
  name: ${device_name}
  friendly_name: ${device_name}
  
bk72xx:
  board: wb2s

logger:

api:

ota:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  manual_ip:
    static_ip: 192.168.0.83
    gateway: 192.168.0.1
    subnet: 255.255.255.0
    dns1: 192.168.0.1

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "${device_name} Fallback Hotspot"

captive_portal:

web_server:

switch:
  - platform: gpio
    pin: PWM4
    inverted: false
    name: "Relay"
    id: relay
    on_turn_on:
      - light.turn_on:
          id: led1
    on_turn_off:
      - light.turn_off:
          id: led1

binary_sensor:
  - platform: status
    name: "Node Status"
    id: system_status

  - platform: gpio
    pin:
      number: PWM5
      inverted: true
      mode:
        input: true
        pullup: true
    name: "Button Input"
    id: button_input
    on_press:
      then:
        - switch.toggle: relay

light:
  - platform: binary
    name: "led1"
    id: led1
    output: led1_output

output:
  - platform: gpio
    pin: PWM1
    id: led1_output
    inverted: true
~~~

Then choose Install, Manual download, finally select UF2 Package format and save to local disk.

## Flashing the generated firmware

This step requires preparing the WB2S to be flashed and installing the flashing tool.

### Hardware preparation

A few basic hardware tools are required:

- USB-TTL adapter
- Soldering iron
- some cables
- a multi-tester

Under REFERENCES below there is a link to the WB2S datasheet.

The first step requires soldering temporary wires from the WB2S to a USB-TTL adapter. Only five wires are
needed: pins 1RX, 1TX, CEN, GND and VBAT. 1RX from the WB2S whould go to Tx on the TTL adapter, 
likewise, 1TX should go to Rx on the other end. 

VBAT should go to 3.3V (Datasheet suggests to use a separate power supply as the TTL 
adapter may not suffice, but I had no problem whith mine). 

Obviously, GND from WB2S should go to GND on the TTL adapter.

CEN should to to GND temporarily while plugging the USB-TTL adapter in order to put the WB2S in 
flashing mode.

### Software flashing tool

~~~
pip install ltchiptool
~~~

To read (download) firmware from the chip:

~~~
ltchiptool flash read BK7231T teckastar_ori
~~~

~~~
To write (flash) new firmware to the chip:
~~~

ltchiptool flash write teclastar1.uf2

### REFERENCES

https://developer.tuya.com/en/docs/iot/wb2s-module-datasheet?id=K9ghecl7kc479
https://docs.libretiny.eu/
