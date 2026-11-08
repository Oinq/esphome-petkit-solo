# ESPHome firmware for the PETKIT Fresh Element Solo

Local, cloud-free firmware for the **PETKIT Fresh Element Solo** pet feeder, built on
[ESPHome](https://esphome.io). It reuses the feeder's own ESP32. No extra board is needed.

> **Status: pre-release (0.1.0), not yet validated on a feeding animal.**
> See [Tested](#tested) before using it.

## Features

- Feeding by portions, stopped by the rotation sensor.
- **Motor protection:** at most one feed at a time, 1–6 portions per request
  (configurable), a stop when the rotation sensor goes silent for 6 s, a 60 s absolute
  limit, a forward/reverse interlock and motor off at boot.
- **Four scheduled meals stored on the feeder.** They keep running when Home Assistant
  is down. Time, portions and on/off are editable from Home Assistant and survive reboots.
- Time from NTP (`pool.ntp.org` by default) and from Home Assistant.
- Result of every feed: `ok`, `no_food_detected` (the motor turned but the optical
  sensor saw no food) or `motor_timeout`. It is also sent as the Home Assistant event
  `esphome.petkit_solo_feed`.
- Physical button feeds the manual portion count.
- Home Assistant entities: *Feed now*, *Manual portions*, *Meal 1–4 time / portions /
  enabled*, *Last feed result*, *Last feed portions*, *Portions today*,
  *Food possibly empty*, Wi-Fi signal, uptime, version, restart.
- Home Assistant action `esphome.<name>_feed` with `portions`.

*Food possibly empty* is inferred from the optical sensor. The feeder has no scale and
cannot measure grams or hopper level.

## Hardware

Main board `D4.2_1_MAIN_PCBA_V1.0`, ESP32 (ESP32-WROOM-32D or ESP32-D0WD depending on
the revision).

| GPIO | Function |
|---|---|
| 0 | Wi-Fi/pairing button (low at power-up = bootloader) |
| 5 | Status LED |
| 14 | Optical food sensor |
| 17 / 18 | Motor reverse / forward |
| 19 | Motor driver enable (nSleep) |
| 27 | Rotation sensor (one pulse per portion) |
| 33 | Sensor power enable |
| 34 | Feed button |
| 1 / 3 | UART TX / RX (used for flashing) |

Other board revisions may differ. Check yours before flashing.

## Installation

The stock firmware is not ESPHome, so the **first flash needs a USB–UART adapter at
3.3 V**. After that, updates are over the air.

1. Open the feeder and disconnect it from power. Remove the batteries if present.
2. Connect the adapter: adapter TX → board `WIFI_RX0`, adapter RX → board `WIFI_TX0`,
   GND → GND and 3.3 V → `3V3`. Never use 5 V on these pins.
3. Hold the Wi-Fi button (GPIO0) while powering up to enter the bootloader.
4. Check the flash size first: `esptool --port /dev/ttyUSB0 --before no-reset flash-id`.
   Units with 4 MB and 8 MB flash exist; the image works on both.
5. **Back up the whole flash first.** It is the only way back. Use the size from step 4
   (`0x400000` for 4 MB, `0x800000` for 8 MB), read it twice and compare the SHA-256:
   `esptool --port /dev/ttyUSB0 --before no-reset --after no-reset read-flash 0x0 0x800000 petkit-solo-original.bin`
6. Flash the factory image from the release (or build it from
   [`example/petkit-solo.yaml`](example/petkit-solo.yaml) with `esphome compile`):
   `esptool --port /dev/ttyUSB0 --before no-reset --after no-reset write-flash 0x0 petkit-solo-factory.bin`.
   No `secrets.yaml` is needed: Wi-Fi is set up on first boot and Home Assistant
   provisions the API key when you adopt the device.

With only RX/TX/GND/3V3 wired, the adapter cannot reset the ESP32, and it may leave the
bootloader between esptool runs. Before each command, power-cycle with the Wi-Fi button
held, and keep it held until esptool connects.

Install over the network from your own configuration with the remote package:

```yaml
packages:
  petkit_solo: github://Oinq/esphome-petkit-solo/petkit-solo.yaml@v0.1.0
```

## Configuration

Override any of these in `substitutions:`:

| Substitution | Default | Meaning |
|---|---|---|
| `name`, `friendly_name` | `petkit-solo`, `PETKIT Solo` | Device names |
| `timezone` | `Europe/London` | Time zone for the schedule |
| `ntp_server_1..3` | `0..2.pool.ntp.org` | NTP servers; use a local one for an offline setup |
| `meal1_time` … `meal4_time` | 07:00, 12:00, 17:00, 21:00 | Initial meal times |
| `default_meal_portions` | `2` | Initial portions per meal |
| `default_manual_portions` | `1` | Initial portions for the button / *Feed now* |
| `max_portions` | `6` | Absolute limit per request |
| `pulse_timeout_ms` | `6000` | Stop if no rotation pulse for this long |
| `hard_timeout` | `60s` | Absolute limit for one feed |

Meals are **disabled** on a new device: enable them in Home Assistant after setting
the times and portions. Initial values only apply on first boot. After that the values set in Home Assistant
are kept.

Without network time (no Internet, no Home Assistant) the schedule does not run until
the clock is set. Manual feeding still works.

## Safety

This firmware drives a motor that feeds an animal. A fault can leave an animal without
food or dispense too much.

- Test with supervision and calibrate grams per portion with your food.
- Watch the *Last feed result* and the `esphome.petkit_solo_feed` events.
- Keep an independent way to notice missed meals.
- The timeouts stop the motor when the rotation sensor fails. They cannot detect a
  jam that still produces pulses, or a broken food chute.

Use at your own risk. See the [licence](LICENSE).

## Tested

| Item | Status |
|---|---|
| Configuration valid on ESPHome 2026.10.2 | ✅ |
| Compiles on ESPHome 2026.10.2 (arduino, ~917 KB, 51 % of the app partition) | ✅ |
| Runs on a Solo (board D4.2) | pending |
| Motor protection in a real failure | pending |
| Second unit | pending |

## Credits

Built on community reverse engineering:

- [rickypr](https://www.reddit.com/r/Esphome/comments/108nqwc/) and
  [pinkpandahug](https://www.reddit.com/r/Esphome/comments/v19c7p/): original ESPHome
  configurations for the Solo.
- [Alexiy2/PetKit-Fresh-Element-Solo](https://github.com/Alexiy2/PetKit-Fresh-Element-Solo):
  GPIO map.
- [foxey](https://gist.github.com/foxey/d9aa8b711ad1a76d3b5271c54302ca09): motor
  protection timeout pattern.
- The [Home Assistant community thread](https://community.home-assistant.io/t/338971).

## Licence

[MIT](LICENSE).
