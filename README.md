# Watering Computer

ESPHome configuration for an ESP32-based 4-zone irrigation controller with Home Assistant integration.

## Overview

Drives an 8-channel relay board to control four irrigation zones, a pressure pump, and a 24V transformer. The transformer relay (CH8) is automatically energized whenever any zone is active, and the pump (CH7) runs whenever a traditional zone (1–3) is active. Power is only drawn during watering.

A daily schedule runs each zone sequentially at configurable times (up to 4 slots per day), with skip conditions for weather, manual override, and a master enable switch. A separate **bleed mode** (légtelenítés) runs zones 1–3 with the pump suppressed to purge air from the lines.

## Hardware

- **MCU:** ESP32 (chip revision 3.1+), ESP-IDF framework
- **Relay board:** 8-channel active-HIGH

| Channel | GPIO   | Function                                          |
|---------|--------|---------------------------------------------------|
| CH1     | GPIO32 | Zone 1 (traditional — uses pump)                  |
| CH2     | GPIO33 | Zone 2 (traditional — uses pump)                  |
| CH3     | GPIO25 | Zone 3 (traditional — uses pump)                  |
| CH4     | GPIO26 | Unused                                            |
| CH5     | GPIO27 | Unused                                            |
| CH6     | GPIO14 | Zone 4 (no pump, transformer only)                |
| CH7     | GPIO12 | Pump (auto: on with traditional zones, off in bleed mode) |
| CH8     | GPIO13 | Transformer power (auto: on with any zone)        |

## Features

- **Aligned 5-minute scheduler with up to 4 slots.** Polls on the next `:00`, `:05`, `:10`… boundary (+10s grace) once SNTP/HA time is valid. Each of the 4 schedule slots has an enable flag and its own hour/minute; any enabled slot whose time matches fires the full sequence (each slot can fire once per day).
- **Sequential zone run.** Each of the 4 zones runs back-to-back for its configured duration (0–60 min, default 10 min; 0 skips the zone).
- **Skip conditions:**
  - Master Enable off
  - Manual Skip Today (auto-resets at midnight)
  - Weather OK off (settable from HA via the `set_weather_ok` service)
- **Auto transformer control.** CH8 follows the count of active zone relays; turns on for the first zone, off after the last.
- **Auto pump control.** CH7 turns on while any traditional zone (1–3) is active; suppressed during bleed mode and idle for zone 4.
- **Bleed mode (légtelenítés).** Runs zones 1–3 sequentially with the pump suppressed to purge air; triggered by a button or HA.
- **Persistent settings.** Schedule slots, zone durations, moisture thresholds, and toggles survive reboots via `restore_value: yes`.
- **Emergency stop button** turns off all zones and the pump immediately.
- **HA events** emitted on start, complete, skip, emergency stop, and bleed start/complete (`esphome.watering_*`, `esphome.bleed_*`).
- **Web UI** on port 80 with HTTP Basic auth, Hungarian labels, and sorted groups (Zones / Schedule / Program / Diagnostics).
- **Wi-Fi roaming.** 802.11k/v (RRM/BTM) enabled for multi-AP wired-backhaul setups.
- **Diagnostics:** uptime, ESP32 temperature, schedule time, active zones, skip reason, last-watered relative time.

## Home Assistant integration

- Time source: `homeassistant` platform (primary) with SNTP fallback, timezone `Europe/Budapest`.
- Service exposed: `esphome.watering_computer_set_weather_ok(weather_clear: bool)` — typically called from an HA automation that watches a forecast/rain sensor.

## Setup

1. Install [ESPHome](https://esphome.io/) (2025.11.0 or newer).
2. Create a `secrets.yaml` next to `watering-computer.yml` with:
   ```yaml
   wifi_ssid: "your-ssid"
   wifi_password: "your-password"
   web_user: "admin"
   web_password: "your-web-password"
   ```
3. Compile and flash:
   ```bash
   esphome run watering-computer.yml
   ```
4. Adopt the device in Home Assistant.

## Configuration

All runtime knobs are exposed as HA entities:

- **Slot 1–4 Hour / Minute** — start times (minute snaps to 5-minute steps)
- **Schedule Slot 1–4 Enabled** — per-slot toggles; defaults: slot 1 on (06:00), 2–4 off
- **Zone 1–4 Duration (min)** — set 0 to skip a zone
- **Moisture Skip / Top-up Threshold (%)** — globals only; not yet wired to sensors
- **Master Enable**, **Manual Skip Today**, **Weather OK**
- Buttons: **Run Full Schedule Now**, **Bleed (Légtelenítés)**, **Emergency Stop**

## Fallback AP

If Wi-Fi fails, the device exposes a captive portal at SSID `Watering-Computer Fallback` using the same Wi-Fi password as the secret.
