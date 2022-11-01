# esphome-ensto
YAML only implementation of Ensto thermostat support for ESPHome.  
Requires ESPHome 2022.10.0 or later.

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

It is recommended to adjust device's own temperature setting so that it is close to wanted temperature. That way any failure in Home Assistant, WiFi, ESP32, bluetooth or this code can be mitigated by turning the ESP32 off. Boost offsets used to control the thermostat have reasonable timeouts to give control back to the thermostat within about 30 minutes.


# Integration to Home-assistant
Add ESPHome integration to Home-assistant from **Configuration - Devices & Services**

Esphome-ensto should be automatically discovered with 1 device and multiple entities.
There is an example how to add sensor to Lovelace UI in *home-assistant/ui-lovelace.yaml* file. It will show reading and statuses from the thermostat in graphs.

## Usage with Thermostat Climate Controller
Simplest way to control the thermostat is to add *climate.ensto1_thermostat* as a Thermostat card to Home Assistant. Turning Heat mode on will set boost modes accordingly to reach set temperature.

Thermostat Climate Controller requires setting external temperature sensor to *esphome-ensto.yaml* by changing *sensor.bedroom_temperature* to the ID of the sensor in Home Assistant that measures the actual temperature in the room where the thermostat is.

    # External temperature sensor from Home Assistant for Climate Control
      - platform: homeassistant
        id: external_temperature_sensor
        entity_id: sensor.bedroom_temperature
        internal: true
        accuracy_decimals: 2
        # Optional filter in case original sensor requires filtering. Note that median filter will make reacting to temperature changes slower.
        #filters:
        #  - median:
        #      window_size: 6
        #      send_every: 1
        #      send_first_at: 1

## Advanced configuration
Thermostat Climate Controller functionality can be adjusted by changing the value of

    heat_deadband: 0.5°C
    heat_overrun: 0.5°C

and by modifying algorithm for setting the boost value.  
Mainly this part

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

Boost value is set on purpose to value that is too high or too low so that temperature reaches target temperature instead of device's thermostat keeping it just below the target because of variations in its internal temperature measurements.

# Observations
## Power modes
The device I have is Ensto Beta 10 BT EB 1000W.  
According to the BT interface specification there are 5 different power modes:  
1 = Floor  
2 = Room  
3 = Combination  
4 = Power  
5 = Force control  

From those it can be set only to Room and Power modes.  
Room mode uses temperature selection wheel from the device to set wanted temperature.  
Power mode uses temperature selection wheel to directly controlling the power cycle of the device between 0-100%, meaning how much heating relay is ON and OFF.  
I tested controlling temperature in both.

BT interface specification talks about Force control mode which could be used to set temperature selection wheel value programmatically, but values read from that bluetooth characteristic didn't match what is specified in the interface specification. I have not tried to use that mode.

## Power control
Boost in Room mode can be set as degrees from -20 to 20 allowing negative boost. Boost in Room mode uses device's internal temperature sensor to decide when and how much to heat to reach the temperature changed with the boost. 

Boost in Power mode is set as 0-100% on top of power output from selection wheel. So negative boost is not allowed. Automation in Power mode would require temperature selection wheel to be turned completely to 0% value so that full power range could be controlled with the boost offset.

From device control point-of-view, using Power mode would give complete control to automation and this code to control the temperature to wanted value. I tested using PID Climate Controller component instead of Thermostat Climate Controller to reach and stay in selected target temperature. I was not able to make it work the way I would have wanted in Room mode, but I assume that in Power mode it could be used. In Room mode the automatic calibration didn't give values that would control temperature reliably. I didn't test it in Power mode.

From risk management point-of-view, using Room mode is safer. When something goes wrong in ESP32, Bluetooth or this code, it usually happens at 2am and bedroom temperature is off by multiple degrees.  
In Room mode all that is needed is to turn ESP32 or Thermostat Climate Controller from Home Assistant OFF and to trust device's own thermostat to take over for it to get to the temperature that is already set with the temperature selection wheel. Boost value set by ESP32 would timeout within 30 minutes.  
In Power mode the mode would first need to be changed to Room mode, ESP32 would need to be turned OFF and then temperature selection wheel would need to be turned from zero to correct temperature.
