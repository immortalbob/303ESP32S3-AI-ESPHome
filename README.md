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
I (232) Board: UUID=ee2f52e2-c6cf-4633-b1e0-a9fb258aea01 SKU=bread-compact-wifi
This SKU matched the Xiaozhi AI open source firmware repository.

### 2. Factory Firmware Analysis
A full flash backup was extracted using [esptool-js](https://espressif.github.io/esptool-js/)
and analyzed using PowerShell string extraction:
```powershell
$bytes = [System.IO.File]::ReadAllBytes("C:\backup.bin")
[System.IO.File]::WriteAllText("C:\strings_out.txt", [System.Text.Encoding]::ASCII.GetString($bytes))
Select-String -Path "C:\strings_out.txt" -Pattern "gpio|i2s|audio|button|led" | Out-File "C:\results.txt"
```
This confirmed the use of `NoAudioCodecSimplex` driver, `AdcButton` for buttons,
`SingleLed` for LEDs, and the source file path `./main/boards/bread-compact-wifi/compact_wifi_board.cc`.

### 3. Audio Pin Confirmation
The INMP441 and MAX98357A GPIO assignments were cross-referenced against the Xiaozhi firmware
source and confirmed working through ESPHome compilation and testing.

### 4. Button Discovery
Buttons were found using ADC scanning — adding temporary ADC sensors to every available GPIO
and monitoring voltage changes while pressing each button:
- GPIO1 dropped to 0V when WiFi button pressed (confirmed)
- GPIO39 changed state when Vol Down pressed (confirmed)
- GPIO40 changed state when Vol Up pressed (confirmed)

The BOOT and EN buttons are hardware pins not accessible via GPIO.

### 5. PIR Pin Discovery
The PIR header OUT pin was found by adding digital binary sensor inputs across all remaining
unidentified GPIO pins and connecting an HC-SR501 PIR sensor. GPIO8 was confirmed when it
changed state in response to motion detection.

### 6. LED Investigation
An exhaustive scan of all available ESP32-S3 GPIO pins (GPIO1-21, GPIO38-42, GPIO48) was
performed as both digital inputs and outputs. No pins were found that corresponded to the
2x red or 1x green LEDs. These LEDs are believed to be hardwired to power rails and not
under ESP32 control.

---

## ESPHome Setup

### Prerequisites
- [ESPHome](https://esphome.io) 2026.5.0 or later
- Home Assistant with:
  - [openWakeWord](https://www.home-assistant.io/integrations/open_wake_word/) configured
  - [Faster-Whisper](https://www.home-assistant.io/integrations/faster_whisper/) for STT
  - A configured Voice Assistant pipeline

### Secrets Required
Add these to your ESPHome `secrets.yaml`:
```yaml
wifi_ssid: "Your WiFi SSID"
wifi_password: "Your WiFi Password"
api_key: "Generate with ESPHome dashboard"
ap_password: "Your fallback AP password"
```

### First Flash
Since this board has a CH340X USB chip, flashing is straightforward:
1. Connect via USB-C
2. No need to hold BOOT or press EN — one-key flash is supported
3. Flash using ESPHome dashboard or CLI

### Multiple Rooms
To deploy to multiple boards, only change these values per device:
- `name:`
- `friendly_name:`
- `api encryption key:` — generate a new one for each device

Everything else including GPIO assignments is identical across all boards of this model.

### Volume Button Automations
The Vol Up and Vol Down buttons are exposed as binary sensors in HA.
Example automation:
```yaml
automation:
  - alias: "Voice Satellite Volume Up"
    trigger:
      - platform: state
        entity_id: binary_sensor.master_bedroom_volume_up
        to: "on"
    action:
      - service: number.set_value
        target:
          entity_id: number.master_bedroom_volume
        data:
          value: "{{ [states('number.master_bedroom_volume') | float + 10, 100] | min }}"
```

### WiFi Button Automations
The WiFi button is fully repurposable. Example — use it to toggle a scene:
```yaml
automation:
  - alias: "WiFi Button Toggle Scene"
    trigger:
      - platform: state
        entity_id: binary_sensor.master_bedroom_wifi_button
        to: "on"
    action:
      - service: scene.turn_on
        target:
          entity_id: scene.bedroom_night
```

### PIR Motion Automations
The PIR sensor is exposed as a motion binary sensor in HA. Example — wake a screen on motion:
```yaml
automation:
  - alias: "PIR Wake Screen"
    trigger:
      - platform: state
        entity_id: binary_sensor.master_bedroom_motion
        to: "on"
    action:
      - service: light.turn_on
        target:
          entity_id: light.bedroom_lights
```

---

## Known Issues / TODO

- [ ] 2x Red LED GPIO pins not identified — believed hardwired to power rail
- [ ] 1x Green LED GPIO pin not identified — believed hardwired as power indicator
- [ ] Volume buttons wired as binary sensors — HA automations needed for actual volume control
- [ ] Blue battery LEDs cannot be disabled in software (TP5400 hardware controlled)

---

## Contributing

If you've identified the LED GPIO pins or any other undiscovered features of this board,
please submit a PR! This was a community reverse engineering effort and further contributions
are welcome.

---

## Credits

Reverse engineered over several days through binary analysis, ADC scanning, digital pin
scanning, PIR sensor testing, and a lot of trial and error. Special thanks to the Xiaozhi
open source firmware project whose boot logs and source code structure made this possible.

---

## License

MIT — use freely, attribution appreciated but not required.
