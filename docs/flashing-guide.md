# Flashing guide: PETKIT Fresh Element Solo to ESPHome

This guide shows how to replace the stock firmware of a PETKIT Fresh Element Solo with
this ESPHome firmware. It is based on two units converted by the author (board
`D4-2_1_MAIN_PCBA_V1.0`, BOM V2.1 and V2.4) and keeps the photos here, so the guide does
not depend on forum posts staying online.

The first flash needs a USB–UART adapter and a soldering iron. After that, updates go
over Wi-Fi.

> **Warning.** You open a mains-powered device and solder on its board. Disconnect the
> power supply and remove the batteries before you start. Use **3.3 V** logic only; 5 V
> on these pads can destroy the ESP32. Everything here is at your own risk.

## 1. What you need

- A USB–UART adapter with a **3.3 V** setting. The author used an FT232RL board
  (AZ-Delivery) with the jumper on 3.3 V.
- A **data** USB cable for the adapter. Many mini-USB cables only charge; if the computer
  does not see the adapter, try another cable first.
- Four thin wires, a soldering iron and solder.
- A computer with [esptool](https://docs.espressif.com/projects/esptool/) (`pip install esptool`).
- The factory image from the [latest release](https://github.com/Oinq/esphome-petkit-solo/releases),
  or your own build of [`example/petkit-solo.yaml`](../example/petkit-solo.yaml).

![FT232RL adapter](images/ftdi-adapter.jpg)

Set the voltage jumper to **3.3 V**. On this adapter the header pins are, from left to
right, `DTR RX TX VCC CTS GND`.

![Adapter voltage jumper on 3.3 V](images/ftdi-jumper-3v3.jpg)

## 2. Open the feeder

Remove the food hopper and open the base. Inside you find the main board standing
upright, the dispensing motor with its rotation sensor, and the DC power jack.

![Feeder base opened](images/open-base-overview.jpg)

![Second unit opened, board lifted](images/second-unit-open.jpg)

Before you unplug anything, take photos of the connectors and how the cables are routed.
They are glued, and you will need to put them back the same way.

![Board mounted, cable routing](images/board-mounted-cable-routing.jpg)

![Board front, in the case](images/board-front-in-case.jpg)

The front of the board has the ESP32 module (ESP32-WROOM-32D, chip ESP32-D0WD), two
buttons, the buzzer and the connectors.

![Board front](images/board-front.jpg)

![Board front, second unit](images/second-unit-board-front.jpg)

## 3. Find the pads

All the pads you need are on the back of the board and are labelled:

| Pad | Use |
|---|---|
| `WIFI_RX0` | ESP32 receive (connect to adapter **TX**) |
| `WIFI_TX0` | ESP32 transmit (connect to adapter **RX**) |
| `GND` (next to `WIFI_TX0`) | Ground |
| `3V3` (next to `DC/BAT_SW`, at the other end) | 3.3 V supply from the adapter |
| `IO0` | GPIO0; tie to GND at power-up to enter the bootloader (alternative to the Wi-Fi button) |

![Pads to use, marked](images/pads-marked.jpg)

![Back of the board, labels readable](images/pads-clean.jpg)

![Back of the board with the version label](images/board-back-label.jpg)

![Back of the board, second unit (BOM V2.1)](images/second-unit-board-back.jpg)

## 4. Solder the wires

Solder one wire to each of `GND`, `WIFI_TX0`, `WIFI_RX0` and `3V3`. In the photos:
turquoise is `GND`, yellow is `WIFI_TX0`, orange is `WIFI_RX0` and red is `3V3`.

![Wires soldered](images/wires-soldered-overview.jpg)

![Solder joints, close-up](images/wires-soldered-closeup.jpg)

Connect them to the adapter, **crossing TX and RX**:

| Board | Adapter |
|---|---|
| `WIFI_RX0` | `TX` |
| `WIFI_TX0` | `RX` |
| `GND` | `GND` |
| `3V3` | `VCC` (jumper on 3.3 V) |

![Board connected to the adapter](images/board-ftdi-connected.jpg)

**Power.** With the feeder's own power supply disconnected, the adapter powers the ESP32
through `3V3`. This worked on both units. If the adapter cannot supply enough current
(esptool cannot connect, or the computer loses the adapter), use the feeder's power
supply instead and **disconnect the `3V3` wire**: only GND, TX and RX. Never connect both
supplies at the same time.

## 5. Enter the bootloader

1. Disconnect the adapter from USB.
2. Hold the feeder's Wi-Fi button (GPIO0), or tie `IO0` to `GND`.
3. Connect the adapter to USB while holding it.
4. Keep holding until esptool says it is connected, then release.

The adapter is not wired to the ESP32 reset line, so esptool cannot reset the chip by
itself. Always use `--before no-reset`. The ESP32 may also leave the bootloader between
two esptool commands; if a command fails with `No serial data received`, repeat steps
1–4 before each command.

Check the connection and the flash size:

```sh
esptool --port /dev/ttyUSB0 --before no-reset chip-id
esptool --port /dev/ttyUSB0 --before no-reset flash-id
```

On Windows the port is `COM3`, `COM5`, etc. Units with **4 MB and 8 MB** flash exist;
`flash-id` prints the size. The factory image works on both.

## 6. Back up the original firmware

This is the only way back to the stock firmware. Read the **whole** flash, with the size
from `flash-id` (`0x400000` for 4 MB, `0x800000` for 8 MB):

```sh
esptool --port /dev/ttyUSB0 --baud 460800 --before no-reset --after no-reset \
  read-flash 0x0 0x800000 petkit-solo-original-1.bin
```

Then enter the bootloader again (section 5), read a second copy to
`petkit-solo-original-2.bin` and compare both SHA-256 hashes. They must be identical.
Keep the backup somewhere safe. If `--baud 460800` gives read errors, use `115200`.

## 7. Flash the factory image

Check the SHA-256 of the downloaded image against the release notes. Enter the
bootloader again and write it at `0x0`:

```sh
esptool --port /dev/ttyUSB0 --baud 460800 --before no-reset --after no-reset \
  write-flash 0x0 petkit-solo-0.1.0-factory.bin
```

Do not run `erase-flash` and do not force `--flash-size`. The command must end with
`Hash of data verified.`

## 8. First boot and Home Assistant

1. Disconnect the adapter (you can leave the wires on the board for later) and put the
   board back, with the connectors and cables as they were.
2. Power the feeder from its normal power supply, **without** pressing the button.
3. Connect to the open Wi-Fi network `petkit-solo-xxxxxx` (the last six digits of the
   MAC address). If the setup page does not open, go to `http://192.168.4.1`, choose your
   network and enter the password. Wi-Fi can also be set up over USB or Bluetooth with
   [Improv](https://www.improv-wifi.com/).
4. Home Assistant discovers the feeder. Adopt it; Home Assistant provisions the API
   encryption key. If adding it fails with "Unable to connect", simply try again: a weak
   Wi-Fi signal can make the first attempt time out.
5. Test **Feed now** with food in the hopper, standing next to the feeder.
6. Scheduled meals are **off** by default. Set the times and portions and enable them in
   the device page.

Later updates go over the air from ESPHome; the UART wires are no longer needed.

## Restore the original firmware

Enter the bootloader (section 5) and write the backup back:

```sh
esptool --port /dev/ttyUSB0 --before no-reset --after no-reset \
  write-flash 0x0 petkit-solo-original-1.bin
```

## Troubleshooting

| Symptom | Check |
|---|---|
| The computer does not see the adapter | Charge-only USB cable; loose mini-USB plug; adapter LED off. If you use a virtual machine, pass the adapter (USB `0403:6001`) through to it. |
| `No serial data received` | ESP32 not in the bootloader: repeat section 5. TX and RX not crossed: swap them. Jumper on 5 V instead of 3.3 V. |
| Works once, next command fails | The ESP32 left the bootloader. Repeat section 5 before each esptool command. |
| Adapter disconnects when the board is attached | Not enough current from the adapter: use the feeder's power supply with only GND, TX and RX. |
| No `petkit-solo-…` network after flashing | Wait 30 s; check that the button was not held at power-up (that enters the bootloader again). |
