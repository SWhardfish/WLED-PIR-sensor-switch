# WLED Arcade Firmware

The entrance controller runs WLED 0.15.1 on the `esp32s3_4M_qspi` target. Arcade preset `222` is a
temporary presentation override. WLED remains responsible for its PIR and time-controlled preset
policy.

## Build

From the `WLED-PIR-sensor-switch` checkout:

```powershell
py -m pip install --user platformio
npm install
npm test
py -m platformio run -e esp32s3_4M_qspi
```

The OTA image is written to:

```text
build_output/release/WLED_0.15.1_ESP32-S3_4M_qspi.bin
```

Do not upload a filesystem image. A firmware-only OTA update leaves the LittleFS partition that
contains `cfg.json`, `wsec.json`, and `presets.json` intact.

## Backup

Before every update, create a directory outside the Git checkout and download these endpoints from
the controller at `192.168.1.42`:

```powershell
curl.exe --fail "http://192.168.1.42/edit?download=/cfg.json" -o cfg.json
curl.exe --fail "http://192.168.1.42/edit?download=/presets.json" -o presets.json
curl.exe --fail "http://192.168.1.42/json/info" -o info.json
curl.exe --fail "http://192.168.1.42/json/state" -o state.json
```

Treat the backup as secret material. `cfg.json` contains the Wi-Fi and MQTT configuration. Confirm
that all four files contain valid JSON and that `presets.json` contains presets `100`, `101`, `102`,
and `222` before continuing.

The expected hardware/configuration preflight is:

- release `ESP32-S3_4M_qspi`, ESP32-S3, 4 MB flash, PSRAM
- 60 RGBW LEDs on GPIO 6
- PIR usermod enabled on GPIO 8 with a 300-second delay
- PIR on-preset `102`, off-preset `100`, schedule fallback enabled
- MQTT broker `192.168.1.57`, device topic `wled/entrance`
- time-controlled presets `100` and `101`
- Arcade preset `222`

## OTA Update

Open `http://192.168.1.42/update`, select only
`WLED_0.15.1_ESP32-S3_4M_qspi.bin`, and start the update. Do not select or upload a filesystem
image. Keep power connected until WLED has rebooted and the web UI is reachable again.

## Validation

After reboot:

1. Re-download `json/info`, `json/state`, `cfg.json`, and `presets.json` and compare the relevant
   hardware, network, timer, usermod, and preset values with the backup.
2. Confirm MQTT reconnects and `wled/entrance/motion` still reports PIR transitions.
3. Start Arcade and confirm preset `222` remains active despite PIR changes.
4. Exit to Mirror and confirm the preset active before Arcade is restored, not a fixed preset.
5. Start Arcade across a configured timer boundary. Confirm `222` stays active, then exit and confirm
   the newly scheduled preset is restored.
6. Test PIR after Arcade has ended and confirm its timeout restores the latest scheduled preset.

The deployed integration was validated through the complete retained mode path:

```text
mirror preset 100 -> arcade preset 222 -> mirror preset 100
```

## Recovery

If WLED does not return after OTA, do not erase the device. Use USB recovery with the same
`esp32s3_4M_qspi` board and partition target. Restore the backed-up JSON files only after the
firmware is stable and only if the LittleFS contents were actually lost.
