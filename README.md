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
