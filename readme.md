<p align="center">
  <img src="/images/wled_logo_akemi.png">
  <a href="https://github.com/Aircoookie/WLED/releases"><img src="https://img.shields.io/github/release/Aircoookie/WLED.svg?style=flat-square"></a>
  <a href="https://raw.githubusercontent.com/Aircoookie/WLED/master/LICENSE"><img src="https://img.shields.io/github/license/Aircoookie/wled?color=blue&style=flat-square"></a>
  <a href="https://wled.discourse.group"><img src="https://img.shields.io/discourse/topics?colorB=blue&label=forum&server=https%3A%2F%2Fwled.discourse.group%2F&style=flat-square"></a>
  <a href="https://discord.gg/QAh7wJHrRM"><img src="https://img.shields.io/discord/473448917040758787.svg?colorB=blue&label=discord&style=flat-square"></a>
  <a href="https://kno.wled.ge"><img src="https://img.shields.io/badge/quick_start-wiki-blue.svg?style=flat-square"></a>
  <a href="https://github.com/Aircoookie/WLED-App"><img src="https://img.shields.io/badge/app-wled-blue.svg?style=flat-square"></a>
  <a href="https://gitpod.io/#https://github.com/Aircoookie/WLED"><img src="https://img.shields.io/badge/Gitpod-ready--to--code-blue?style=flat-square&logo=gitpod"></a>

  </p>

# Welcome to my project WLED! ✨

A fast and feature-rich implementation of an ESP8266/ESP32 webserver to control NeoPixel (WS2812B, WS2811, SK6812) LEDs or also SPI based chipsets like the WS2801 and APA102!

## ⚙️ Features
- WS2812FX library with more than 100 special effects  
- FastLED noise effects and 50 palettes  
- Modern UI with color, effect and segment controls  
- Segments to set different effects and colors to user defined parts of the LED string  
- Settings page - configuration via the network  
- Access Point and station mode - automatic failsafe AP  
- Up to 10 LED outputs per instance
- Support for RGBW strips  
- Up to 250 user presets to save and load colors/effects easily, supports cycling through them.  
- Presets can be used to automatically execute API calls  
- Nightlight function (gradually dims down)  
- Full OTA software updateability (HTTP + ArduinoOTA), password protectable  
- Configurable analog clock (Cronixie, 7-segment and EleksTube IPS clock support via usermods) 
- Configurable Auto Brightness limit for safe operation  
- Filesystem-based config for easier backup of presets and settings  

## 💡 Supported light control interfaces
- WLED app for [Android](https://play.google.com/store/apps/details?id=com.aircoookie.WLED) and [iOS](https://apps.apple.com/us/app/wled/id1475695033)
- JSON and HTTP request APIs  
- MQTT   
- E1.31, Art-Net, DDP and TPM2.net
- [diyHue](https://github.com/diyhue/diyHue) (Wled is supported by diyHue, including Hue Sync Entertainment under udp. Thanks to [Gregory Mallios](https://github.com/gmallios))
- [Hyperion](https://github.com/hyperion-project/hyperion.ng)
- UDP realtime  
- Alexa voice control (including dimming and color)  
- Sync to Philips hue lights  
- Adalight (PC ambilight via serial) and TPM2  
- Sync color of multiple WLED devices (UDP notifier)  
- Infrared remotes (24-key RGB, receiver required)  
- Simple timers/schedules (time from NTP, timezones/DST supported)  

## Custom modifications in this fork

This private deployment fork is based on WLED 0.15.1 and modifies the
[`PIRsensorSwitch`](usermods/PIR_sensor_switch/usermod_PIR_sensor_switch.h) usermod for two related
requirements:

1. PIR motion must return to the correct time-controlled preset when its timeout ends.
2. An external presentation, currently the MMEntrance Arcade, must temporarily own the visible
   preset without replacing WLED's schedule or PIR policy.

### Schedule-aware PIR restore

Stock `PIRsensorSwitch` can apply a fixed `off-preset` when motion ends. That is insufficient for an
installation whose normal preset changes through WLED timers: a morning, daytime, evening, or off
schedule can change while PIR motion is active.

This fork adds two persisted usermod settings:

- `use-time-preset`: enables schedule-aware behavior after the PIR off-timer expires.
- `use-schedule-fallback`: restores the most recent time-controlled preset when enabled; when
  disabled, `off-preset` remains the fallback.

The restore order after PIR timeout is:

1. Restore `lastScheduledPreset` when both options are enabled and a scheduled preset is known.
2. Apply `off-preset` when time-preset handling is enabled but schedule fallback is disabled.
3. Use the legacy off-preset, previous playlist, previous preset, temporary preset, or lights-off
   behavior.

WLED timer, sunrise, and sunset macros now apply presets with `CALL_MODE_TIME_PRESET`. This dedicated
call mode lets the usermod distinguish real scheduler changes from MQTT, web UI, button, PIR, and
other preset changes. External preset `222`, for example, can no longer become the remembered
scheduled preset merely because it was selected through MQTT.

### Presentation override

The presentation override is controlled through WLED's existing JSON API, including the MQTT JSON
API topic. It introduces no additional server or protocol.

Start an override and atomically select its preset:

```json
{
  "PIRsensorSwitch": {
    "presentation": "start",
    "preset": 222
  },
  "ps": 222
}
```

Release the override:

```json
{
  "PIRsensorSwitch": {
    "presentation": "restore"
  }
}
```

Override behavior:

- The first `start` snapshots the current non-PIR preset as the restoration baseline.
- Repeated `start` commands are idempotent and reassert the presentation preset without overwriting
  the baseline. MMEntrance uses this to repair missed state after MQTT or WLED reconnects.
- PIR transitions cannot replace the presentation preset while the override is active.
- A WLED timer, sunrise, or sunset event during the override updates `lastScheduledPreset`, then the
  presentation preset is immediately reasserted.
- `restore` releases the override and asynchronously applies the latest scheduled preset.
- A restore-pending guard prevents WLED's immediate state callback from misclassifying the outgoing
  presentation preset as the new baseline.
- The override is intentionally runtime-only. A controller reboot clears it, allowing retained
  MMEntrance mode to reassert the correct state after reconnect.

For MMEntrance, the MQTT topic is `wled/entrance/api`, Arcade uses preset `222`, and Mirror or Guest
releases the override instead of forcing a fixed normal preset.

### Production target

The deployed controller has been queried and updated successfully with this repository's
`esp32s3_4M_qspi` environment:

- WLED 0.15.1, release `ESP32-S3_4M_qspi`
- ESP32-S3 at 240 MHz
- 4 MB flash with PSRAM
- 60 RGBW LEDs on GPIO 6
- PIR input on GPIO 8 with a 300-second timeout
- PIR on-preset `102`, off-preset `100`
- time-controlled presets `100` and `101`
- Arcade preset `222`

The environment explicitly defines `USERMOD_PIRSWITCH`; without that build flag, the modified
header is present in the source tree but the usermod is not instantiated in the firmware.

### Build and deployment

```powershell
py -m pip install --user platformio
npm install
npm test
py -m platformio run -e esp32s3_4M_qspi
```

The OTA application image is generated at:

```text
build_output/release/WLED_0.15.1_ESP32-S3_4M_qspi.bin
```

Upload only this application image through WLED's `/update` page. Do not upload a filesystem image:
the LittleFS partition contains `cfg.json`, `wsec.json`, and `presets.json`. Always back those files
up before an update. See [docs/WLED-ARCADE-FIRMWARE.md](docs/WLED-ARCADE-FIRMWARE.md) for the complete
backup, OTA, rollback, and validation procedure.

## �📲 Quick start guide and documentation

See the [documentation on our official site](https://kno.wled.ge)!

[On this page](https://kno.wled.ge/basics/tutorials/) you can find excellent tutorials and tools to help you get your new project up and running!

## 🖼️ User interface
<img src="/images/macbook-pro-space-gray-on-the-wooden-table.jpg" width="50%"><img src="/images/walking-with-iphone-x.jpg" width="50%">

## 💾 Compatible hardware

See [here](https://kno.wled.ge/basics/compatible-hardware)!

## ✌️ Other

Licensed under the EUPL v1.2 license  
Credits [here](https://kno.wled.ge/about/contributors/)!

Join the Discord server to discuss everything about WLED!

<a href="https://discord.gg/QAh7wJHrRM"><img src="https://discordapp.com/api/guilds/473448917040758787/widget.png?style=banner2" width="25%"></a>

Check out the WLED [Discourse forum](https://wled.discourse.group)!  

You can also send me mails to [dev.aircoookie@gmail.com](mailto:dev.aircoookie@gmail.com), but please, only do so if you want to talk to me privately.  

If WLED really brightens up your day, you can [![](https://img.shields.io/badge/send%20me%20a%20small%20gift-paypal-blue.svg?style=flat-square)](https://paypal.me/aircoookie)


*Disclaimer:*   

If you are prone to photosensitive epilepsy, we recommended you do **not** use this software.  
If you still want to try, don't use strobe, lighting or noise modes or high effect speed settings.

As per the EUPL license, I assume no liability for any damage to you or any other person or equipment.  

