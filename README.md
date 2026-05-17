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
| LEDs | 4x Blue (TP5400 battery indicator), 1x Red, 1x Green |
| PIR | Header exposed (GND, OUT, VCC) |

---

## Confirmed GPIO Pinout

This pinout was determined through a combination of:
- Boot log analysis of the factory Xiaozhi firmware
- Binary firmware extraction and hex analysis
- ADC scanning across all available GPIO pins
- Physical PCB trace inspection

| Function | GPIO | Notes |
|---|---|---|
| INMP441 WS (mic clock) | GPIO4 | I2S |
| INMP441 SCK (mic bit clock) | GPIO5 | I2S |
| INMP441 SD (mic data) | GPIO6 | I2S |
| MAX98357A DIN (speaker data) | GPIO7 | I2S |
| MAX98357A BCLK (speaker bit clock) | GPIO15 | I2S |
| MAX98357A LRC (speaker word select) | GPIO16 | I2S |
| Volume Down button | GPIO39 | INPUT_PULLUP, inverted |
| Volume Up button | GPIO40 | INPUT_PULLUP, inverted |
| WiFi button | GPIO1 | INPUT_PULLUP, inverted |
| BOOT | Hardware | Flash mode, not GPIO controllable |
| EN | Hardware | Reset pin, not GPIO controllable |
| PIR OUT | Unknown | To be determined |
| Red/Green LEDs | Unknown | Scanning incomplete |
| 4x Blue LEDs | N/A | TP5400 hardware controlled, not ESP32 |

---

## Hardware Notes

### What's NOT on this board variant
- **No OLED display** — The Xiaozhi firmware attempts to initialize an SSD1306 display and fails. 
  A sibling SKU exists with the display populated. Your board is the "compact" variant without it.
- **No camera** — Camera support exists in the firmware but no header is present.

### LEDs
The 4 blue LEDs are driven directly by the TP5400 battery management IC and indicate charge level. 
They **cannot** be controlled via software. The only way to silence them is physical 
(tape, nail polish, or desoldering).

The red and green LEDs have not been mapped to GPIO pins. If you figure them out, please submit a PR!

### Battery
The TP5400 supports external 3.7V LiPo batteries via the PH2.0 connector. Pay attention to 
polarity — positive and negative are marked on the PCB.

### PSRAM
This board uses **octal PSRAM** (N16R8). The `psram: mode: octal` setting is mandatory. 
Using the wrong mode will cause boot failures or instability.

### Flash Mode
`CONFIG_ESPTOOLPY_FLASHMODE_QIO: "y"` is required in sdkconfig_options. Without it the board 
may experience subtle instability or boot issues.

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

### Volume Button Automations
The Vol Up and Vol Down buttons are exposed as binary sensors in Home Assistant. 
Wire them to automations to control whatever you like. Example automation for volume:

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

---

## Known Issues / TODO

- [ ] Red and green LED GPIO pins not yet identified
- [ ] PIR header GPIO pin not yet identified  
- [ ] Volume buttons wired as binary sensors — HA automations needed for actual volume control
- [ ] Blue battery LEDs cannot be disabled in software

---

## Contributing

If you've identified the LED GPIO pins, PIR GPIO pin, or any other undiscovered features 
of this board, please submit a PR! This was a community reverse engineering effort and 
further contributions are welcome.

---

## Credits

Reverse engineered over several days through binary analysis, ADC scanning, and a lot of 
trial and error. Special thanks to the Xiaozhi open source firmware project whose boot logs 
and source code structure made this possible.

---

## License

MIT — use freely, attribution appreciated but not required.
