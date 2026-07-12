# WLEDkitty

WLEDkitty is [WLED](https://github.com/wled/WLED) with a small, consistent Retia
personality, built for [Retia](https://retia.io) hardware (2024 DEF CON badge,
Bluetooth Nugget, USB Nugget, Pusheen, …):

- **Boots straight into the Pride 2015 rainbow** on the onboard RGB (and any external strip).
- **Live status** on the device's screen (color TFT on the badge, OLED on the Nuggets).
- **Face-button control**: A = on/off, B = next effect, D-pad = brightness / color / effect.

Everything else is stock WLED — same web UI, same JSON API, same effects and palettes.

This is a fork of `wled/WLED`; the Retia changes are intentionally small and live on the
[`wledkitty`](../../tree/wledkitty) branch so they stay diffable against upstream. Base
commit: `wled/WLED @ bc2c80d`. Licensed under the **EUPL-1.2** (same as WLED) — see
[`../LICENSE`](../LICENSE).

## What's patched

| File | Change |
|---|---|
| `wled00/wled.h`, `wled00/wled.cpp`, `wled00/FX.h` | `DEFAULT_BOOT_FX` (Pride two-knob boot default); `WLED_LAUNCHER_GUEST` (FS-less build for the badge SD launcher). |
| `wled00/button.cpp` | Retia d-pad handlers — UP/DOWN = brightness, LEFT/RIGHT = hue; `WLED_NUGGET_S2_DPAD` variant for the 4-button USB Nugget. |
| `usermods/ST7789_display/` | Color-TFT status usermod (ILI9341 support, registration + AP-flicker fixes, live preview bar). |
| `usermods/usermod_v2_four_line_display_ALT/` | OLED status usermod (`FLD_FLIP_DEFAULT` guard). |
| `platformio_override.ini` | One build env per Retia device (see below). |
| `retia/partitions-shared*.csv` | Partition tables for the badge SD-launcher guest envs. |

## Devices / build envs

Each device is a PlatformIO env in [`../platformio_override.ini`](../platformio_override.ini):

| Env | Device | Chip | Notes |
|---|---|---|---|
| `badge-dual` | 2024 DEF CON badge | ESP32-S3 8MB | dual LED bus + ILI9341 TFT |
| `nugget` | Bluetooth Nugget | ESP32-S3 4MB | GPIO10 ears + SSD1306 OLED |
| `usb-nugget` | USB Nugget | ESP32-S2 | GPIO12 + SH1106 OLED, 4-button d-pad |
| `pusheen` | Pusheen cat lamp | ESP8266 | 4 LEDs on D1/GPIO5, no display |
| `badge-launcher-v2` | badge (SD-launcher guest) | ESP32-S3 8MB | FS-less app image |

## Building

WLED's ESP32 platforms reject Python 3.14 — build with a **Python 3.13** venv:

```bash
pip install platformio          # into a py3.13 venv
pio run -e usb-nugget           # or badge-dual / nugget / pusheen / ...
```

Output is `.pio/build/<env>/firmware.bin`.

### Merge to a flashable 0x0 image

ESP8266 (`pusheen`): `firmware.bin` is already a complete 0x0 image — flash it directly.

ESP32 (`--flash-mode dio` is universally safe; the badge boot-loops on a QIO header):

```bash
BOOT_APP0=$(find ~/.platformio/packages/framework-arduinoespressif32*/tools/partitions/boot_app0.bin | head -1)
esptool --chip esp32s2 merge-bin -o wledkitty-usb-nugget.factory.bin \
  --flash-mode dio --flash-freq 80m --flash-size 4MB \
  0x1000 .pio/build/usb-nugget/bootloader.bin \
  0x8000 .pio/build/usb-nugget/partitions.bin \
  0xE000 "$BOOT_APP0" \
  0x10000 .pio/build/usb-nugget/firmware.bin
```

(ESP32-S3 bootloader offset is `0x0`, not `0x1000`; adjust `--chip`/`--flash-size` per device.)

## Gotchas

- **Pride needs two knobs.** Set BOTH `-D DEFAULT_MODE=63` and `-D DEFAULT_BOOT_FX=63` — a
  global `effectCurrent` overrides the segment mode at boot otherwise.
- **`BTNPIN` compacts over `-1`.** WLED builds its button list skipping disabled (`-1`) pins,
  so leading `-1`s shift the real buttons to lower indices. Order `BTNPIN` for the *compacted*
  index and match the `button.cpp` `switch` to it (see `WLED_NUGGET_S2_DPAD`).
- **Keep LED/data pins out of `BTNPIN`.** WLED's default button is GPIO0; if an LED bus is on
  GPIO0 it goes dark. Always set `BTNPIN` explicitly.
