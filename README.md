# esphome-ensto
YAML only implementation of Ensto thermostat support for ESPHome.

# Installation
## Docker-compose
Install Docker CE and docker-compose (e.g. https://techviewleo.com/install-and-use-docker-compose-on-linux-mint/)

Copy or clone this repository to PC.  
Start docker-compose in root folder of the repository with Terminal command:

    docker-compose up

## Configuration for ESPHome
Open dashboard with web browser as instructed by docker-compose output (e.g. http://0.0.0.0:6052)

Open **Secrets** menu from dashboard and fill values according to your environment. These values are used in esphome-ensto.yaml. Add any home-assistant keys etc. in similar fashion to the Secrets and esphome-ensto.yaml if needed.

    wifi_ssid: "REPLACEME"  
    wifi_password: "REPLACEME"  
    ensto1_mac: "RE:PL:AC:EM:YM:AC"
  
**Note:** Device MAC address can be found from Ensto's own official mobile App. Refer to user manual for how to connect the App with the thermostat.

Change **board** from *esphome-ensto.yaml* according to your ESP32 board. List of possible values are available in https://docs.platformio.org/en/latest/platforms/espressif32.html#boards  
Leave **platform** as "ESP32".  
For example, correct values for DOIT DEVKIT V1 board are:

    platform: ESP32  
    board: esp32doit-devkit-v1

## Install ESPHome to ESP32
Attach ESP32 board to PC with an USB cable. Initial installation needs to be done using USB, but after that updates can be done using OTA with WiFi.  
Select **Update** or **Install** (from **...** menu) for **esphome-ensto** and select **Plug into this computer**.  

In case device is not listed, you might need to install proper driver according to these instructions: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/establish-serial-connection.html  
For example DOIT DEVKIT V1 needs CP210x driver.  
There might be a need to make USB port visible to Docker by adjusting **devices** configuration from *docker-compose.yaml*.  
For example:

    devices:
        - /dev/ttyUSB0:/dev/ttyUSB0

**Note**: Restarting docker is needed if *docker-compose.yaml* is modified. Stop it with Ctrl+C and start it again.

All subsequent updates can be done via Wifi by selecting **Wirelessly** from the install menu.

## Connecting with the thermostat
Thermostat needs to be set to pairing mode for it to accept client connection from ESPHome. That is done in a same way as when connecting the official mobile App.  

1. Close official mobile App, as thermostat will allow only one connected client at the time. Connected client will block other clients from connecting. It is safest to restart the thermostat to make sure old connection doesn't exist anymore.
2. Press bluetooth pairing button from the thermostat for few seconds until blue led starts to blink.  
3. Power up ESP32. It will automatically detect that the thermostat is in pairing mode and it will save the pairing information permanently for future connecions. After that ESP32 will automatically reconnect when ever it, or the thermostat, is restarted.


# Integration to Home-assistant
Add ESPHome integration to Home-assistant from **Configuration - Devices & Services**

Esphome-ensto should be automatically discovered with 1 device and multiple entities.
There is an example how to add sensor to Lovelace UI in *home-assistant/ui-lovelace.yaml* file. It will show reading and statuses from the thermostat in graphs.

## Automation example
In this automation example we will use one external temperature sensor to measure a real room temperature and control the thermostat to keep the real room temperature within ±1°C of selected target temperature. For that we use the Boost feature of the thermostat.

First let's create adjustable target temperature value.
Add following to *configuration.yaml* file of your home-assistant instance

    input_number:
      ensto1_target_external_temperature:
        name: Target external temperature
        min: 15
        max: 26
        step: 0.1
        unit_of_measurement: '°C'
        icon: mdi:home-thermometer-outline

Then add 2 dummy sensors for calculating temperature differences. First one calculates difference between room temperature from external sensor and target temperature. Second one calculates the value we need to send to the ESPHome for setting the boost.

    sensor:
      - platform: template
        sensors:
          ensto1_target_temperature_diff:
            unit_of_measurement: '°C'
            entity_id:
              - sensor.bedroom_temperature
            value_template: >-
              {% set room = states('sensor.bedroom_temperature')|float %}
              {% set target = states('input_number.ensto1_target_external_temperature')|float %}
              {% if room == 0.0 %}
                {{ 0 }}
              {% else %}
                {{ (room - target)|round(2) }}
              {% endif %}
    # Temperature difference between target temperature and real room temperature.
    # Difference with limits -4.9..5.0 decrees is scaled to 1..100. Ready to be sent as fan
    # speed percentage to ESPHome. ((val + 5) / 10 * 100)
    # Zero value is handled as uninitialized condition in automation.
          ensto1_temperature_boost_diff:
            unit_of_measurement: '°C'
            entity_id:
              - sensor.bedroom_temperature
            value_template: >-
              {% set room = states('sensor.bedroom_temperature')|float %}
              {% set target = states('input_number.ensto1_target_external_temperature')|float %}
              {% if room == 0.0 %}
                {{ 50 }}
              {% else %}
                {{ (max(-4.9, min((target - room)|round(2), 5)) + 5) * 10 }}
              {% endif %}

**Note:** Boost temperature and other values are sent to ESPHome using range 0-100, as esphome-ensto.yaml uses Fan control for setting them. Fan supports only percentage values between 0-100%. So the formula above converts temperature difference between -5.0 and 5.0 to values between 0 and 100. That is only caused by limitation in Fan control. Real values could be sent if some other control would be used, but at the moment I don't know how to do that and using Fan as a workaround is "good enough".

Home-assistant needs to be restarted after *configuration.yaml* has been modified. Restart button can be found from **Configuration - Settings - Server Controls** menu.

Add new Gauge card to Home-assistant UI for controlling the target temperature. Clicking the Gauge card will show a slider for changing the value.

    type: gauge
    entity: input_number.ensto1_target_external_temperature
    min: 15
    max: 26
    severity:
      green: 19
      yellow: 15
      red: 22
    needle: true

We can also add temperature difference between real room temperature and the target temperature to a history graph.

    type: history-graph
    entities:
      - entity: sensor.ensto1_temperature_calibration_value
        name: Calibration value
      - entity: sensor.ensto1_target_temperature_diff
        name: Target diff
      - entity: sensor.ensto1_temperature_boost_offset
        name: Boost offset
    hours_to_show: 24
    refresh_interval: 0

And the target room temperature to a graph with other temperatures

    type: history-graph
    entities:
      - entity: sensor.ensto1_room_temperature
        name: Temperature
      - entity: sensor.ensto1_target_temperature
        name: Target
      - entity: sensor.bedroom_temperature
        name: External
      - entity: input_number.ensto1_target_external_temperature
        name: External target
    hours_to_show: 24
    refresh_interval: 0

Now we can add new automation. Here is copy-paste from my automation.yaml, but it can be inputted using the UI. 

    - id: '1645461355693'
      alias: Set Ensto boost value
      description: ''
      trigger:
      - platform: state
        entity_id: sensor.ensto1_target_temperature_diff
      condition:
      - condition: or
        conditions:
      - condition: numeric_state
        entity_id: sensor.ensto1_target_temperature_diff
        above: '1'
      - condition: numeric_state
        entity_id: sensor.ensto1_target_temperature_diff
        below: '-1'
      action:
      - service: fan.set_percentage
        data:
          percentage: '{{ states.sensor.ensto1_temperature_boost_diff.state|int }}'
        target:
          entity_id: fan.ensto1_temperature_boost_offset_set
      - delay:
        hours: 0
        minutes: 60
        seconds: 0
        milliseconds: 0
      mode: single

Above is "Single-mode" automation, meaning that only one instance of it is run at a time. The 60 minute delay at the end makes sure that automation is run only once per hour.  
It uses State trigger for *sensor.ensto1_target_temperature_diff* value.  
Condition for trigger is that Numeric state of the value is above 1 OR below -1. *Or* condition type must be added first and then 2 *Numeric state* conditions for above and below values.  
Action uses following YAML (use edit in YAML)

    service: fan.set_percentage
    data:
      percentage: '{{ states.sensor.ensto1_temperature_boost_diff.state|int }}'
    target:
      entity_id: fan.ensto1_temperature_boost_offset_set

Second action is *Wait for time to pass* with 60 minutes as a value.
