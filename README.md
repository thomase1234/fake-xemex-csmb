## fake-xemex-csmb
Modbus Client that can simulate a Xemex CSMB by simulating a ModBus-RTU Server - based on M5Stack ATOM Lite ESP32

## Goal

This project allows you to set the charge speed of a Shell Recharge Advanced 3.0 charging point by emulating a Xemex CSMB.

## How is this working ?

ESPhome device acting as a Xemex CSMB by simulating a Modbus RTU Slave/Client/Server that can be polled by a master (e.g. a EV Wallbox like Shell Recharge Advanced 3.0) and delivers IREGs and HREGs which can be controlled arbitrarily. I'm doing this using [Home Assistant](https://www.home-assistant.io/) and [EVCC](https://evcc.io/en/). [EVCC](https://evcc.io/en/) has been installed as add-on on [Home Assistant](https://www.home-assistant.io/).