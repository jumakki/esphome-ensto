# esphome-ensto

YAML only implementation of Ensto thermostat support for ESPHome.  
Requires ESPHome 2022.10.0 or later.

## Usage

Implementation supports different ways to control thermostats

1. I want to change and schedule temperatures from Home Assistant just like I can with Ensto's App with Boost offsets. See [example 1](#1-setting-boost-offset-from-home-assistant-automation).
2. I want to control thermostat's heating based on external temperature sensor in Home Assistant, because thermostat's internal temperature sensor varies too much and real room temperature is not stable. See [example 2](#2-usage-with-thermostat-climate-controller).

In addition also following uses are possible

3. I want to track in Home Assistant how much electricity the thermostat uses and how much it costs with current electricity price. See [example 3](#3-setting-electricity-price).
4. I want to show status of different properties from the thermostat in Home Assistant. See [Exposed Entities](#exposed-entities) below.

### Exposed Entities

Following entities are exposed and usable from Home Assistant for reading values.

| Name                                          | ID                                                 |
|-----------------------------------------------|----------------------------------------------------|
| Connection status (ON/OFF)                    | binary_sensor.ensto1_connection_status             |
| Boost enabled (ON/OFF)                        | binary_sensor.ensto1_boost_status                  |
| Heating relay active (ON/OFF)                 | binary_sensor.ensto1_relay_state                   |
| Measured room temperature                     | sensor.ensto1_room_temperature                     |
| Measured floor temperature                    | sensor.ensto1_floor_temperature                    |
| Target temperature in thermostat              | sensor.ensto1_target_temperature                   |
| Temperature boost minutes left                | sensor.ensto1_temperature_boost_minutes_left       |
| Temperature boost offset                      | sensor.ensto1_temperature_boost_offset             |
| Temperature calibration value                 | sensor.ensto1_temperature_calibration_value        |
| Cumulative electricity price for current hour | sensor.ensto1_electricity_price_current_hour       |
| Total electricity price for previous hour     | sensor.ensto1_electricity_price_previous_hour      |
| Max Heating Power of thermostat               | sensor.ensto1_max_heating_power                    |
| Electricity price per kWh                     | sensor.ensto1_price_per_kwh                        |
| Currency ID (€, $, SEK, NOR, RUB)             | sensor.ensto1_currency_id                          |
| Power consumption current hour (kWh)          | sensor.ensto1_power_consumption_current_hour_kwh   |
| Power consumption previous hour (kWh)         | sensor.ensto1_power_consumption_previous_hour_kwh  |
| Power consumption ratio current hour          | sensor.ensto1_power_consumption_ratio_current_hour |
| Power consumption ratio previous hour         | sensor.ensto1_power_consumption_ratio_previous_hour|

### Exposed Services

Following services are exposed and usable from Home Assistant for setting new values.

|ID                                 |Parameters        |Value range                  |
|-----------------------------------|------------------|-----------------------------|
|set_ensto1_temperature_boost_offset|boost_offset      |-20.00 - 20.00 °C            |
|                                   |length_minutes    |0 - 300 minutes              |
|set_ensto1_temperature_calibration |temperature_offset|-5.0 - 5.0 °C                |
|set_ensto1_kwh_price               |energy_unit_id    |1=€, 2=SEK, 3=NOK, 4=RUB, 5=$|
|                                   |price_per_kwh     |0.00 - 655.35                |
|set_ensto1_max_power               |max_power         |0 - 65535 W                  |

### Exposed Climate Controller

Following Climate Thermostat Controller is exposed for controlling thermostat based on external temperature sensor.

|Name             |ID                       |
|-----------------|-------------------------|
|Ensto1 Thermostat|climate.ensto1_thermostat|

## Installation

## Docker-compose

Install Docker CE and docker-compose (e.g. <https://techviewleo.com/install-and-use-docker-compose-on-linux-mint/>)

Copy or clone this repository to PC.  
Start docker-compose in root folder of the repository with Terminal command:

```shell
docker-compose up
```

### Configuration for ESPHome

Open dashboard with web browser as instructed by docker-compose output (e.g. <http://0.0.0.0:6052>)

Open ***Secrets*** menu from dashboard and fill values according to your environment. These values are used in [esphome-ensto.yaml](esphome-ensto.yaml). Add any home Assistant keys etc. in similar fashion to the Secrets and esphome-ensto.yaml if needed.

```yaml
wifi_ssid: "REPLACEME"  
wifi_password: "REPLACEME"  
ensto1_mac: "RE:PL:AC:EM:YM:AC"
```

> ***Note:*** Device MAC address can be found from Ensto's own official mobile App. Refer to user manual for how to connect the App with the thermostat.

Change `board` from [esphome-ensto.yaml](esphome-ensto.yaml) according to your ESP32 board. List of possible values are available in <https://docs.platformio.org/en/latest/platforms/espressif32.html#boards>  
Leave `platform` as "ESP32".  
For example, correct values for DOIT DEVKIT V1 board are:

```yaml
platform: ESP32  
board: esp32doit-devkit-v1
```

### Install ESPHome to ESP32

Attach ESP32 board to PC with an USB cable. Initial installation needs to be done using USB, but after that updates can be done using OTA with WiFi.  
Select ***Update*** or ***Install*** (from ***...*** menu) for ***esphome-ensto*** and select ***Plug into this computer***.  

In case device is not listed, you might need to install proper driver according to these instructions: <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/establish-serial-connection.html>  
For example DOIT DEVKIT V1 needs CP210x driver.  
There might be a need to make USB port visible to Docker by adjusting `devices` configuration from [docker-compose.yaml](docker-compose.yaml).  
For example:

```yaml
devices:
    - /dev/ttyUSB0:/dev/ttyUSB0
```

> ***Note:*** Restarting docker is needed if [docker-compose.yaml](docker-compose.yaml) is modified. Stop it with Ctrl+C and start it again.

All subsequent updates can be done via Wifi by selecting **Wirelessly** from the install menu.

### Connecting with the thermostat

Thermostat needs to be set to pairing mode for it to accept client connection from ESPHome. That is done in a same way as when connecting the official mobile App.  

1. Close official mobile App, as thermostat will allow only one connected client at the time. Connected client will block other clients from connecting. It is safest to restart the thermostat to make sure old connection doesn't exist anymore.
1. Press bluetooth pairing button from the thermostat for few seconds until blue led starts to blink.  
1. Power up ESP32. It will automatically detect that the thermostat is in pairing mode and it will save the pairing information permanently for future connecions. After that ESP32 will automatically reconnect when ever it gets disconnected or it, or the thermostat, is restarted.

It is recommended to adjust thermostat's own temperature setting so that it is close to wanted temperature. That way any failure in Home Assistant, WiFi, ESP32, Bluetooth or this code can be mitigated by turning the ESP32 off. Boost offsets used to control the thermostat have reasonable timeouts to give control back to the thermostat within about 30 minutes.

## Integration to Home Assistant

Add ESPHome integration to Home Assistant from **Configuration - Devices & Services**

Esphome-ensto should be automatically discovered with 1 device and multiple entities.
There is an example how to add sensor to Lovelace UI in [home-assistant/ui-lovelace.yaml](home-assistant/ui-lovelace.yaml) file. It will show reading and statuses from the thermostat in graphs.

## Usage examples

### 1. Setting Boost offset from Home Assistant automation

Following automation reduces room temperature by 2°C every day between 23:00 and 06:00 using thermostat's Boost offset with negative value.

Automation is triggered every minute. Trigger could be also change in thermostat's temperature or something else. Boost length is set to 55 minutes so that set boost offset remains also in case of a short connection breakage between Home Assistant and ESPHome, or ESPHome and thermostat. Additional condition is used to write new boost offset to thermostat only every 5 minutes to reduce writes.

Automation below can be copy-pasted to Home Assistant by creating new automation, and selecting `Edit in YAML` from Options.

> ***Note:*** The full name of Boost offset service depends on the name of used ESPHome node. In example below it is `esphome-ensto` causing full service name to be `esphome.esphome_ensto_set_ensto1_temperature_boost_offset`.

> ***Note:*** Boost value should not be set when Thermostat Climate Controller is turned ON like in [example 2](#2-usage-with-thermostat-climate-controller) as any Boost set by this automation will be overwritten by Climate Controller. Only one method of controlling the thermostat should be used at the time.

```yaml
alias: Night time temperature drop
trigger:
  - platform: time_pattern
    minutes: "1"
condition:
  - condition: numeric_state
    entity_id: sensor.ensto1_temperature_boost_minutes_left
    below: 50
action:
  - choose:
      - conditions:
          - condition: time
            before: "06:00:00"
            after: "23:00:00"
        sequence:
          - service: >-
              esphome.esphome_ensto_set_ensto1_temperature_boost_offset
            data:
              boost_offset: -2
              length_minutes: 55
      - conditions: []
        sequence:
          - service: >-
              esphome.esphome_ensto_set_ensto1_temperature_boost_offset
            data:
              boost_offset: 0
              length_minutes: 55
mode: single
```

### 2. Usage with Thermostat Climate Controller

To control thermostat based on external temperature sensor, add `climate.ensto1_thermostat` as a Thermostat card to Home Assistant. Turning Heat mode ON will set boost modes accordingly to reach set temperature.

Thermostat Climate Controller requires setting external temperature sensor to [esphome-ensto.yaml](config/esphome-ensto.yaml) by changing `sensor.bedroom_temperature` to the ID of the temperature sensor in Home Assistant that measures the actual temperature in the room where the thermostat is.

```yaml
# External temperature sensor from Home Assistant for Climate Control
    - platform: homeassistant
      id: external_temperature_sensor
      entity_id: sensor.bedroom_temperature
      internal: true
      accuracy_decimals: 2
      # Optional filter in case original sensor requires filtering. Note that median filter will make reacting to temperature changes slower.
      #filters:
      #  - median:
      #      window_size: 7
      #      send_every: 1
      #      send_first_at: 1
```

> ***Note:*** Don't set Boost offset in automation, like in [example 1](#1-setting-boost-offset-from-home-assistant-automation), when Thermostat Climate Controller is turned ON. Instead control Thermostat Climate Controller directly with automation.

#### Advanced configuration

Thermostat Climate Controller functionality can be adjusted by changing the values of

```yaml
heat_deadband: 0.5°C
heat_overrun: 0.5°C
```

and by modifying algorithm for setting the boost value.  
Mainly this part:

```c
// finally overcompensate needed boost so that target temperature is actually reached
// These formulas could be improved
if (thermos_diff < -1 * id(ensto1_thermostat).heat_overrun()) {
    // above and not close to target (e.g. >0.5 from target) -> big negative boost to turn heating completely off
    needed_boost += -5;
} else if (thermos_diff > id(ensto1_thermostat).heat_deadband()) {
    // below and not close to target (e.g. <0.5 from target) -> boost more than needed
    needed_boost += 2;
} else {
    // close to target -> double the diff and try to keep it here
    needed_boost += thermos_diff * 1.5;
}
```

Boost value is set on purpose to value that is too high or too low so that temperature reaches target temperature instead of device's thermostat keeping it just below the target, because of variations in its internal temperature measurements.

## 3. Setting electricity price

Electricity price can be tracked internally in Home Assistant, but thermostat supports also reading the price of consumed electricity directly from the thermostat.

Current electricity price can be sent to thermostat, for example, with following automation, which sets day price at 07:00 to 0.31€/kWh and night price at 22:00 to 0.28€/kWh. Thermostat supports prices with only 2 decimals. Currency is Euro (€) with `energy_unit_id` value 1 (See [Exposed services](#exposed-services) for other options).

> ***Note:*** The full name of kWh price service depends on the name of used ESPHome node. In example below it is `esphome-ensto` causing full service name to be `esphome.esphome_ensto_set_ensto1_kwh_price`.

```yaml
alias: Electricity price
description: ""
trigger:
  - id: day
    platform: time
    at: "07:00:00"
  - id: night
    platform: time
    at: "22:00:00"
condition: []
action:
  - variables:
      day_price: 0.31
      night_price: 0.28
      price: >
        {{ day_price if trigger.id == 'day' else night_price }}
  - service: esphome.esphome_ensto_set_ensto1_kwh_price
    data:
      energy_unit_id: 1
      price_per_kwh: "{{ price }}"
mode: single
```

Amount and price of consumed electricity for current hour can be read from `sensor.ensto1_power_consumption_current_hour_kwh` and `sensor.ensto1_electricity_price_current_hour`, and total of previous hour from `sensor.ensto1_power_consumption_previous_hour_kwh` and `sensor.ensto1_electricity_price_previous_hour`.

## Observations

### Power modes

The device I have is **Ensto Beta 10 BT EB 1000W**.  
According to the BT interface specification there are 5 different power modes:  
1 = Floor  
2 = Room  
3 = Combination  
4 = Power  
5 = Force control  

From those it can be set only to Room and Power modes.  
Room mode uses temperature selection wheel from the device to set wanted temperature.  
Power mode uses temperature selection wheel for directly controlling the power cycle of the device between 0-100%, meaning the ratio of heating relay being ON.  
I tested controlling temperature in both.

BT interface specification talks about Force control mode which could be used to set temperature selection wheel value programmatically, but values read from that bluetooth characteristic didn't match what is specified in the interface specification. I have not tried to use that mode.

### Power control

Boost in Room mode can be set as degrees from -20°C to 20°C allowing negative boost. Boost in Room mode uses device's internal temperature sensor to decide when and how much to heat to reach the temperature changed with the boost.

Boost in Power mode is set as 0-100% on top of power output from selection wheel. So negative boost is not possible. Automation in Power mode would require temperature selection wheel to be turned completely to 0% value so that full power range could be controlled with the boost offset.

From device control point-of-view, using Power mode would give complete control to automation and this code to control the temperature to wanted value. I tested using [PID Climate Controller](https://esphome.io/components/climate/pid.html) component instead of [Thermostat Climate Controller](https://esphome.io/components/climate/thermostat.html) to reach and stay in selected target temperature. I was not able to make it work the way I would have wanted in Room mode, but I assume that in Power mode it could be used. In Room mode the [PID autotuning](https://esphome.io/components/climate/pid.html#autotuning) didn't give values that would control temperature reliably. I didn't test it in Power mode.

From risk management point-of-view, using Room mode is safer. When something goes wrong in ESP32, Bluetooth or this code, it usually happens at 2am and bedroom temperature is off by multiple degrees.  
In Room mode all that is needed is to turn ESP32 or Thermostat Climate Controller from Home Assistant OFF and to trust device's own thermostat to take over for it to get to the temperature that is already set with the temperature selection wheel. Boost value set by ESP32 would timeout within 30 minutes.  
In Power mode the mode would first need to be changed to Room mode, ESP32 would need to be turned OFF and then temperature selection wheel would need to be turned from zero to correct temperature.

### Reliability

Connection to Home Assistant and/or Ensto keeps dropping frequently when log output is open in ESPHome. Especially when viewed through Wifi. With USB it seems to be more stable. So it is a good idea to close ESPHome’s log view when not needed.

Location of ESP32 should be close enough (3-4 meters max?) from the thermostat.
