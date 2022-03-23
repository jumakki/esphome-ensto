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
