# esphome-ensto
YAML only implementation of Ensto thermostat support for ESPHome.

# Installation
## Docker-compose
Install Docker CE and docker-compose (e.g. https://techviewleo.com/install-and-use-docker-compose-on-linux-mint/)

Copy or clone this repository to PC.  
Start docker-compose in root folder of the repository with Terminal command
> docker-compose up

## Configure
Open dashboard with web browser as instructed by docker-compose output (e.g. http://0.0.0.0:6052)

Open 'Secrets' from dashboard and fill values according to your environment. These values are used in enphome-ensto.yaml
> wifi_ssid: "REPLACEME"  
> wifi_password: "REPLACEME"  
> ensto1_mac: "RE:PL:AC:EM:YM:AC"

Change 'board:' from esphome-ensto.yaml according to your ESP32 board. List of possible values are available in https://docs.platformio.org/en/latest/platforms/espressif32.html#boards  
Leave 'platform:' as 'ESP32'.  
I have DOIT DEVKIT V1 board, so correct values for me are:
> platform: ESP32  
> board: esp32doit-devkit-v1

## Install to ESP32
Attach ESP32 board to PC with an USB cable. Initial installation needs to be done using USB, but after that updates can be done using OTA with WiFi.  
Select '**Update**' or '**Install**' (from ... menu) for **esphome-ensto** and select **Plug into this computer**.  

In case device is not listed, you might need to install proper driver according to these instructions: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/establish-serial-connection.html  
For example DOIT DEVKIT V1 needs CP210x driver.  
There might be a need to make USB port visible to Docker by adjusting 'devices:' configuration from *docker-compose.yaml*  
For example:
>         devices:
>             - /dev/ttyUSB0:/dev/ttyUSB0
All subsequent updates can be done via Wifi by selecting **Wirelessly** from the install menu

