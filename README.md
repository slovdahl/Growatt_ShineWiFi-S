# Growatt_ShineWiFi
Firmware replacement for Growatt ShineWiFi-S (serial), ShineWiFi-X (USB) or custom build sticks (ESP8266/ESP32).

# How to install

* Download a precompiled release from [here](https://github.com/otti/Growatt_ShineWiFi-S/releases)

Or

* Checkout this repo
* Setup the IDE of your choice
    * **Recommended:** For platformio just open the project folder and choose the correct env for your hardware
    * For the original Arduino IDE follow the instruction in the main ino [file](https://github.com/otti/Growatt_ShineWiFi-S/blob/master/SRC/ShineWiFi-ModBus/ShineWiFi-ModBus.ino) (esp8266 only)
* Rename and adapt [Config.h.example](https://github.com/otti/Growatt_ShineWiFi-S/blob/master/SRC/ShineWiFi-ModBus/Config.h.example) to Config.h with your compile time settings

After you obtained an image you want to flash:

* Flash to an esp32/esp8266 ([details](https://github.com/otti/Growatt_ShineWiFi-S/blob/master/Doc/)).
* Connect to the setup wifi called GrowattConfig (PW: growsolar) and configure the firmware via the webinterface at http://192.168.4.1
* If you need to reconfigure the stick later on you have to either press the ap button (configured in Config.h) or reset the stick twice within 10sec

## Features
Implemented Features:
* Built-in simple Webserver
* The inverter is queried using Modbus Protocol
* The data received will be transmitted by MQTT to a server of your choice.
* The data received is also provided as JSON
* Show a simple live graph visualization  (`http://<ip>`) with help from highcharts.com
* It supports convenient OTA firmware update (`http://<ip>/firmware`)
* It supports basic access to arbitrary modbus data
* It tries to autodected which stick type to use
* Wifi manager with own access point for initial configuration of Wifi and MQTT server (IP: 192.168.4.1, SSID: GrowattConfig, Pass: growsolar)
* Currently Growatt v1.20, v1.24 and v3.05 protocols are implemented and can be easily extended/changed to fit anyone's needs
* TLS support for esp32
* Debugging via Web and Telnet

Not supported:
* It does not make use the RTC or SPI Flash of these boards.
* It does not communicate to Growatt Cloud at all.

## Supported sticks/microcontrollers
* ShineWifi-S with a Growatt Inverter connected via serial (Modbus over RS232 with level shifter)
* ShineWifi-X with a Growatt Inverter connected via USB (USB-Serial Chip from Exar)
* Wemos-D1 with a Growatt Inverter connected via USB (USB-Serial Chip: CH340)
* NODEMCU V1 (ESP8266) with a Growatt Inverter connected via USB (USB-Serial Chip: CH340)
* ShineWifi-T (untested, please give feedback)
* Lolin32 (ESP32) with a Growatt Inverter connected via USB

I tested several ESP8266-boards with builtin USB-Serial converters so far only boards with CH340 do work (CP21XX and FTDI chips do not work). Almost all ESP8266 modules with added 9-Pin Serial port and level shifter should work with little soldering via Serial.

See the short descriptions to the devices (including some pictures) in the "Doc" directory.

## Supported Inverters
* Growatt 1000-3000S 
* Growatt MIC 600-3300TL-X (Protocol 124 via USB/Protocol 120 via Serial)
* Growatt MID 3-25ktl3-x (Protocol 124 via USB)
* Growatt MOD 3-15ktl3-x-xh (Protocol 120 via USB)
* Growatt MOD 12KTL3-X (Protocol 124 via USB)
* Growatt MID 25-40ktl3-x (Protocol 120 via USB)
* Growatt SPH 4000-10000ktl3-x BH (Protocol 124 via Serial)
* And others ....

## Modbus Protocol Versions
The documentation from Growatt on the Modbus interface is available, search for "Growatt PV Inverter Modbus RS485 RTU Protocol" on Google.

The older inverters apparently use Protocol v3.05 from year 2013.
The newer inverters apparently use protocol v1.05 from year 2018.
There is also a new protocol version v1.24 from 2020. (used with SPH4-10KTL3 BH-UP inverter)


## JSON Format Data
For IoT applications the raw data can now read in JSON format (application/json) by calling `http://<ip>/status`

## Homeassistant configuration


This will put the inverter on the energy dashboard.

```yaml
mqtt:
  sensor:
    - state_topic: "energy/solar"
      unique_id: "growatt_wr_total_production"
      name: "Growatt.TotalGenerateEnergy"
      unit_of_measurement: "kWh"
      value_template: "{{ float(value_json.TotalGenerateEnergy) | round(1) }}"
      device_class: energy
      state_class: total_increasing
      json_attributes_topic: "energy/solar"
      last_reset_topic: "energy/solar"
      last_reset_value_template: "1970-01-01T00:00:00+00:00"
      payload_available: "1"
      availability_mode: latest
      availability_topic: "energy/solar"
      availability_template: "{{ value_json.InverterStatus }}"
```


To extract the current AC Power you have to add a sensor template.

```yaml
template:
  - sensor:
      - name: "Growatt inverter AC Power"
        unit_of_measurement: "W"
        state: "{{ float(state_attr('sensor.growatt_inverter', 'OutputPower')) }}"
```

To send commands you can use the MQTT Publish service. Note that the datetime commands are currently only available on the 1.24 modbus version.

```yaml
service: mqtt.publish
data:
  qos: "1"
  topic: energy/solar/command/datetime/get
  payload_template: |
    {
      "correlationId": "ha-datetime-get"
    }
```

```yaml
service: mqtt.publish
data:
  qos: "1"
  topic: energy/solar/command/datetime/set
  payload_template: |
    {
      "correlationId": "ha-datetime-set",
      "value": "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
    }
```

To receive responses to commands you can use templates.

```yaml
- trigger:
    platform: mqtt
    topic: energy/solar/result
    value_template: "{{ value_json.correlationId }}"
    payload: "ha-datetime-get"
  sensor:
    name: Growatt - Inverter date/time
    state: "{{ trigger.payload_json.value }}"
    attributes:
      success: "{{ trigger.payload_json.success }}"
      message: "{{ trigger.payload_json.message }}"
```

```yaml
- trigger:
    platform: mqtt
    topic: energy/solar/result
    value_template: "{{ value_json.correlationId }}"
    payload: "ha-datetime-set"
  binary_sensor:
    name: Growatt - Inverter set date/time
    state: "{{ trigger.payload_json.success }}"
    attributes:
      message: "{{ trigger.payload_json.message }}"
```

You could create an automation triggered by a binary sensor on the success JSON attribute if you want to be notified when a command has failed for example.

## Debugging

If you turned on `ENABLE_WEB_DEBUG` in the Config.h (see Config.h.example) there is a debug site under `http://<ip>/debug`. You can turn on `ENABLE_TELNET_DEBUG` to get the debug messages via a telnet client. `telnet <ip>`

To enable even more messages, take a look to `DEBUG_MODBUS_OUTPUT`.

## Change log

See [here](CHANGELOG.md)

## Acknowledgements

This arduino sketch will replace the original firmware of the Growatt ShineWiFi stick.

This project is based on Jethro Kairys work on the Modbus interface
https://github.com/jkairys/growatt-esp8266

Some keywords:

ESP8266, ESP-07S, Growatt 1000S, Growatt 600TL, ShineWifi, Arduino, MQTT, JSON, Modbus, Rest
