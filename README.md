# 303ESP32S3-AI v2.3 ESPHome Configuration
## The Undocumented Xiaozhi AI Voice Board — Reverse Engineered

This repository contains a fully working ESPHome configuration for the **303ESP32S3-AI v2.3** board,
sold on eBay and AliExpress as a "Xiaozhi AI Voice Chat" module. This board has no official
English documentation, no schematic, and no ESPHome support — so we made our own.

---

## Board Identification

If you have this board, you'll see these markings on the PCB:

- `esp32s3-ai_v2.3`
- `303esp32ai2`
- `5437314r_p759_251206`
- FCC ID: `2bb77-esps3-32`
- Board size: 39x45.5mm
- Original firmware SKU: `bread-compact-wifi` (Xiaozhi AI v2.0.3, ESP-IDF v5.5)

The board ships running Chinese-language Xiaozhi AI firmware. If your board speaks Chinese when
powered on, you have the right board.

---

## Hardware Specifications

| Component | Details |
|---|---|
| MCU | ESP32-S3-WROOM-1 N16R8 |
| Flash | 16MB |
| PSRAM | 8MB Octal (OPI) |
| Microphone | INMP441 I2S MEMS |
| Amplifier | MAX98357A I2S Class-D |
| USB | CH340X (one-key flash, no manual reset needed) |
| Battery | TP5400 management chip, PH2.0 connector |
| Speaker | PH2.0 connector (included in kit) |
| Buttons | 5x (Vol Up, Vol Down, WiFi, BOOT, EN) |
| LEDs | 4x Blue (TP5400 battery indicator), 2x Red, 1x Green |
| PIR | Header exposed (GND, OUT, VCC) — HC-SR501 compatible |

---

## Confirmed GPIO Pinout

This pinout was determined through a combination of:
- Boot log analysis of the factory Xiaozhi firmware
- Binary firmware extraction and hex analysis
- ADC scanning across all available GPIO pins
- Digital input scanning across all available GPIO pins
- Physical PCB trace inspection

| Function | GPIO | Notes |
|---|---|---|
| INMP441 WS (mic clock) | GPIO4 | I2S |
| INMP441 SCK (mic bit clock) | GPIO5 | I2S |
| INMP441 SD (mic data) | GPIO6 | I2S |
| MAX98357A DIN (speaker data) | GPIO7 | I2S |
| MAX98357A BCLK (speaker bit clock) | GPIO15 | I2S |
| MAX98357A LRC (speaker word select) | GPIO16 | I2S |
| PIR sensor OUT | GPIO8 | HC-SR501 compatible, INPUT |
| Volume Down button | GPIO39 | INPUT_PULLUP, inverted |
| Volume Up button | GPIO40 | INPUT_PULLUP, inverted |
| WiFi button | GPIO1 | INPUT_PULLUP, inverted, repurposable |
| BOOT | Hardware | Flash mode, not GPIO controllable |
| EN | Hardware | Reset pin, not GPIO controllable |
| 2x Red LEDs | Unknown | Not found after exhaustive GPIO scan |
| 1x Green LED | Unknown | Not found after exhaustive GPIO scan |
| 4x Blue LEDs | N/A | TP5400 hardware controlled, not ESP32 |

---

## Hardware Notes

### Buttons
- **Vol Up / Vol Down** — exposed as binary sensors in HA, wire to automations for volume control or any other purpose
- **WiFi button** — in original Xiaozhi firmware this triggered WiFi provisioning. In ESPHome it is fully repurposable via HA automations — use it for anything you like: mute, scene change, presence toggle, etc.
- **BOOT** — hardware flash button, holds GPIO0 low to enter flash mode. Not controllable in firmware.
- **EN** — hardware reset button. Not controllable in firmware.

### PIR Sensor
The PIR header (GND, OUT, VCC) is compatible with the HC-SR501 and similar 3.3V PIR sensors.
The OUT pin is confirmed on **GPIO8**. The HC-SR501 has two adjustment potentiometers:
- **Sensitivity** — adjust before soldering if possible
- **Time delay** — minimum ~3 seconds, set to preference

Allow 30-60 seconds after boot for the HC-SR501 to warm up before it begins detecting reliably.

### LEDs
The **4 blue LEDs** are driven directly by the TP5400 battery management IC and indicate charge
level. They cannot be controlled via software. The only way to silence them is physical
(tape, nail polish, or desoldering).

The **2 red LEDs and 1 green LED** were not found after exhaustive scanning of all available
ESP32-S3 GPIO pins as both inputs and outputs. They are believed to be hardwired to power rails
rather than controlled by the ESP32. The green LED appears to be a simple power indicator
(always on when USB connected). If you identify these pins, please submit a PR!

### What's NOT on this board variant
- **No OLED display** — The Xiaozhi firmware attempts to initialize an SSD1306 display and fails.
  A sibling SKU exists with the display populated. Your board is the "compact" variant without it.
- **No camera** — Camera support exists in the firmware but no header is present on this variant.

### PSRAM
This board uses **octal PSRAM** (N16R8). The `psram: mode: octal` setting is mandatory.
Using the wrong mode will cause boot failures or instability.

### Flash Mode
`CONFIG_ESPTOOLPY_FLASHMODE_QIO: "y"` is required in sdkconfig_options. Without it the board
may experience subtle instability or boot issues after extended uptime.

---

## How We Figured This Out

This board came with zero English documentation. Here's how the GPIO pinout was reverse engineered:

### 1. Board Identification
The board was identified by powering it on and capturing the serial boot log, which contained:
