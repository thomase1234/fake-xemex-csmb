## fake-xemex-csmb

A Modbus client that emulates a Xemex CSMB by simulating a Modbus-RTU server, designed for the M5Stack ATOM Lite ESP32.

## Project Purpose

This project enables you to control the charging speed of a Shell Recharge Advanced 3.0 charging station by emulating a Xemex CSMB device.

## How It Works

An ESPHome device acts as a Xemex CSMB by simulating a Modbus RTU slave/server. It can be polled by a master device (such as a Shell Recharge Advanced 3.0 EV wallbox) and provides IREGs and HREGs that can be set as needed. This setup is managed using [Home Assistant](https://www.home-assistant.io/) and [EVCC](https://evcc.io/en/), with EVCC installed as an add-on in Home Assistant.

VERY IMPORTANT : The ESPHome device cannot directly set the desired charge speed. It can only simulate the current load on the grid. By simulating a current which higher than the grid capacity, the charger will instruct the car to decrease the current. And by simulating a current which is lower than the grid capacity, the charger will instruct the car to increase the consumption.

For example:
- I have a 40A - single phase home connection.
- By trial and error, I've determined that the charger doesn't increase or decrease the charging speed when I simulate a current of 37.8 A. 
- When I simulate a load of 41 A, you'll see that the charging speed will decrease over time.
- When I simulate a load of 34 A, you'll see that the charging speed will increase over time.


## Interesting info

- My house has a single phase - 40 A connection to the power grid. The house also has a photovoltaic installation that delivers up to 5kW peak. My automation tries to optime power consumption from the sun.
- [In a sister project](https://github.com/thomase1234/esphome-modbus-client-xemex-csmb), I've pulled all data from my Xemex CSMB.
- The real CSMB has 1 CT connected to CT1. Using my custom modbus client, I could see that the register for CT1 contained the actual current. The other to registers ( CT2 and CT3 ) had a non-changing value, which I took over in the ESPHome config.
- Immediately after booting, the Shell Recharge 3.0 Wallbox first requests the Device Code register (0x4002). It expects '20802' as a response. If not, it'll continue retrying.
- The Shell Recharge 3 Wallbox requests the 3 CT registers every 2 seconds.
- You have to know what the max Current ( in Amps ) setting is on your Wallbox. In my case, this was set to 37.8 Amps. As soon as the fake CSMB starts reporting values higher than 37.8 Amps, my charging would decrease. Set the input_number main_maximal_current to the correct value.
- The wallbox refuses to limit the current to a value lower than 6A. If you try to do so, the wallbox will set its limit to 0A and charging will stop (even if the car supports charging at a lower current).
- I've asked Shell to lower my minimal charging speed from 8 A to 6 A. The charging will only start if I simulate a load of 31.8 or lower. 

## Data flows

![Data Flows](pictures/Charger_setup.drawio.png "Data Flows")

1. This is the cloud based integration that pulls the State of Charge (SOC or battery level for EVs) and other data from the car. Users can view this data by looking in the Smartphone App which comes with their cars.
2. [Home Assistant](https://www.home-assistant.io/) has integrations for a lot of car brands to pull this information from the Cloud API offered by the Car Manufacturer. The most important field which you'll need is the SOC ( battery level for EVs ).
3. [Home Assistant](https://www.home-assistant.io/) also needs to know what the current power is that the house is currently consuming from the grid. In most cases, this is a positive number which indicates the amount of Watts the house is consuming. With batteries and/or a PV installation on a sunny day, it's possible that this number turns negative. This indicates the house is producing power and delivering the excess power back to the grid. I'm using a [P1 meter from homewizard](https://www.homewizard.com/nl-be/shop/wi-fi-p1-meter-2/). This meter is connected to the P1 port of my smart digital meter. Alternatively, you can use a device like a [Shelly 3EM](https://www.shelly.com/products/shelly-3em?srsltid=AfmBOoq-uEEL-WgyDaEj-tSgtqfD_INxVcu32BPsVuvu-_LCJsli0l13) to measure the same.
4. On [Home Assistant](https://www.home-assistant.io/), I've installed [EVCC](https://evcc.io/en/) as an add-on. I've been using [HA_AS_EVCC_SOURCE.md](https://github.com/marq24/ha-evcc/blob/main/HA_AS_EVCC_SOURCE.md) from [marq24](https://github.com/marq24/).
This will give EVCC enough information to calculate what the charging power should be. 
5. Since I cannot directly set the charge power of a Shell Recharge Advanced 3.0 charge station, I'll be following a different approach. 
```
chargers:
  - name: HOME.charger
    type: custom
    maxcurrentmillis: # setter
      source: http
      uri: http://127.0.0.1:8123/api/services/input_number/set_value
      method: POST
      headers:
        - Authorization: Bearer eyaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
        - Content-Type: application/json
      body: '{"entity_id":"input_number.evcc_desired_current","value":${maxcurrentmillis}}'
      timeout: 2s
      cache: 0s
```
This will set the home assistant input_number.evcc_desired_current to the desired current. Of course, you have to define this input number first in your home assistant config.

6. The sensor.ct1current is automatically calculated based on the desired charging speed and the current grid load. You can find the configuration in [modbus_server_xemex.yaml](/modbus_server_xemex.yaml)
This sensor is directly pulled used by the esphome device.

7. This device is a [M5Stack ATOM Lite ESP32](https://shop.m5stack.com/products/atom-lite-esp32-development-kit) with a [ RS485 to TTL Converter Unit](https://shop.m5stack.com/products/rs485-module). Both items can be connected to eachother. Afterwards, it's very easy to connect the 4 wires which are currently connected to the load balancing module to this RS485 module. The 4 wires are Ground, + , A and B. The ESP32 will be powered using the ground and +. No need for extra power.
The ESPhome config can be found in the [m5stack-modbus-server-xemex.yaml](/m5stack-modbus-server-xemex.yaml)