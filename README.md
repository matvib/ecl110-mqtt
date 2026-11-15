# ECL110 MQTT

A lightweight Python service that bridges a Danfoss ECL Comfort 110 controller (via Modbus RTU) to Home Assistant using MQTT Auto-Discovery.

![Home Assistant dashboard for the ECL110](images/ha-dashboard.png)
*The ECL110 device in Home Assistant — Mode, Room Setpoint and Slope are writable; temperatures, pump, state and Based On are read-only.*

## Features
* **Home Assistant Auto-Discovery:** Instantly creates a device and sensors in HA without manual YAML configuration.
* **Real-Time Monitoring:** Reads Outdoor, Room, Flow, and Return temperatures, accumulated outdoor, pump status, operating state, heating cut-out, summer standby and the control reference (Based On: Outdoor/Room).
* **Two-Way Control:** Mode (`select`), Room Setpoint (`number`, 15–25 °C, whole degrees) and heating-curve Slope (`number`, 0.1–4.0, 0.1 steps) are writable from Home Assistant.
* **Robust Reconnection:** Handles Modbus read failures and MQTT drops gracefully.
* **One-Command Install:** `setup.sh` asks for serial port, MQTT broker and base topic, then installs a hardened systemd service.

## Hardware Requirements
* Danfoss ECL Comfort 110 (Tested on FW 1.08)
* USB to RS485 Adapter 
* A machine to run the script 

### Wiring
![USB-RS485 adapter wired to the ECL110 terminals](images/wiring.jpg)
*RS-485 (Modbus RTU, 19200 8E1) on the ECL110's **Wieland DST85/4** connector. Pinout (front view): pin 1 = GND, pin 2 = D−/B, pin 3 = D+/A, pin 4 = +5 V. Wire the adapter **A/D+ → pin 3**, **B/D− → pin 2**, **GND → pin 1**; leave pin 4 unconnected (its +5 V rating is unconfirmed). Pinout per [Ingramz/ecl110](https://github.com/Ingramz/ecl110) — verify against your board before wiring.*

![Danfoss ECL Comfort 110 controller](images/controller.jpg)
*ECL Comfort 110, App 130, FW 1.08.*

## Installation

```bash
git clone https://github.com/matvib/ecl110-mqtt.git
cd ecl110-mqtt
sudo ./setup.sh
```

The installer asks for:

| Step | Setting | Default |
|---|---|---|
| Serial | USB adapter (picked from `/dev/serial/by-id/`), Modbus unit ID, baud rate | first adapter, `5`, `19200` |
| MQTT | broker host, port, username, password | `localhost:1883`, anonymous |
| Home Assistant | base topic, device name, discovery prefix, poll interval, log level | `ecl110`, `ECL110`, `homeassistant`, `60` s, `INFO` |

It then creates a system user `ecl110` (added to the serial port's group — `dialout`, or `plugdev` on setups that use it — for port access), installs the program and a venv to `/opt/ecl110-mqtt`, writes `/etc/ecl110-mqtt/config.json`, and starts the `ecl110-mqtt` systemd service. Re-running `sudo ./setup.sh` offers the current settings as defaults.

The **base topic** prefixes every MQTT topic (`<base>/status`, `<base>/sensor/<key>/state`, `<base>/cmd/{mode,room_setpoint,slope,refresh}`) and doubles as the HA device identifier, so give each controller its own if you run more than one bridge.

Useful commands:

```bash
journalctl -u ecl110-mqtt -f       # follow the log
sudo nano /etc/ecl110-mqtt/config.json && sudo systemctl restart ecl110-mqtt
sudo ./uninstall.sh                        # remove the service (asks before deleting config)
```

For development you can skip the installer: environment variables (or a `.env` file) override `config.json`, e.g. `ECL110_PORT`, `ECL110_UNIT`, `MQTT_HOST`, `MQTT_USER`, `MQTT_PASS`, `ECL110_BASE_TOPIC`, `LOG_LEVEL`. Point `ECL110_CONFIG` at a different config file if needed.

## Updating from GitHub
`deploy.sh` pulls the latest commit and, only if something changed, re-runs `setup.sh -y` (keeps your config, reinstalls, restarts):

```bash
./deploy.sh
```

To poll automatically, add a systemd timer that runs `deploy.sh` every few minutes (`ecl110-deploy.service` + `ecl110-deploy.timer` with `OnUnitActiveSec=5min`). `deploy.sh` needs passwordless sudo for that; `echo "$USER ALL=(root) NOPASSWD: /home/$USER/ecl110-mqtt/setup.sh -y" | sudo tee /etc/sudoers.d/ecl110-deploy` (adjust the path).

## Current Status
Verified on hardware (ECL Comfort 110, App 130, FW 1.08):

* [x] MQTT auto-discovery and availability
* [x] Monitoring: S1–S4 temps, accumulated outdoor, pump, state, heating cut-out, summer standby, Based On
* [x] Write Mode (`select`) — all five modes confirmed
* [x] Write Room Setpoint (`number`, integer °C, PNU 11180)
* [x] Write heating-curve Slope (`number`, 0.1–4.0)
* [x] `setup.sh` installer and `deploy.sh` GitHub updates

### Register notes
* Room setpoint is written to **PNU 11180** (Modbus 11179, "desired room temperature"), integer degrees — **not** the ×10 readout at 11229, which is read-only and rejects writes (Modbus exception 4).
* Heating-curve slope is **PNU 11175** (Modbus 11174).
* Ingramz's register numbers are raw Modbus addresses; this code's PNUs are `address + 1`.

## Acknowledgments
A massive thank you to [Ingramz/ecl110](https://github.com/Ingramz/ecl110) for documenting the Modbus registers (PNUs) for this controller. Their register map made the read/write logic of this bridge possible!