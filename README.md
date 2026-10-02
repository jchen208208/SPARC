# SPARC

<img width="485" height="400" alt="SPARC" src="https://github.com/user-attachments/assets/b5467583-4feb-4fdb-be22-021c79f3054c" />

SPARC (Spotify Proximity and Remote Control) is a gesture controller for music. You wave a hand over it to skip tracks, pause, or change the volume, without touching a screen or a button. We built it for times when looking at your phone is inconvenient or unsafe, mainly adjusting music while driving.

The current version is an MVP: an ESP32 on a custom PCB, a time-of-flight distance sensor, an 8-LED strip and a LiPo battery, all inside a 3D-printed case we designed in CAD.

## How it works

SPARC pairs over Bluetooth Low Energy as a HID media remote, the same kind of device as the play/pause button on a pair of headphones. It never talks to Spotify. Each gesture sends a standard media key (next, previous, play/pause, volume up, volume down) to the operating system, and the OS passes it to whatever app is playing.

So there is no app to install and no Spotify login, and you don't need Premium. It works on iOS, Android, macOS, Windows and Linux, and with any media app: Spotify, Apple Music, YouTube or a podcast player. Like headphone buttons, it keeps working with the screen off and the phone in your pocket.

The HID report descriptor only declares media keys. Off-the-shelf ESP32 HID libraries present themselves as keyboards, and iOS hides the on-screen keyboard while any keyboard is connected, so you couldn't type in any app with SPARC paired.

## Gestures

The VL53L0X sensor points up and watches a zone from about 4 cm out to 30 cm. Each gesture is a pass, where your hand moves through the zone and out again, or a hold, where it stops inside it.

- One pass skips to the next track.
- Two quick passes go to the previous track.
- Holding your hand still for a moment toggles play/pause.
- Three quick passes enter volume mode.

Next and previous fire about 0.7 seconds after your last pass, because SPARC waits to see whether another pass is coming. In volume mode, the spot where your hand settles becomes the midpoint. Raise your hand above it to turn the volume up and lower it to turn the volume down; the volume keeps stepping while you stay off-center. To leave volume mode, pull your hand out of the zone or flick it up quickly.

If something stays still within a metre of the sensor for five seconds, like a car roof or a shelf, SPARC treats it as a back wall and ends the zone just short of it. With nothing in range, it falls back to the fixed 30 cm.

## Lights

The LED strip animates each gesture. A swipe runs one way for next and the other way for previous, the whole strip pulses for play/pause, and a looping sweep shows volume changes. A separate status LED flashes on play/pause.

On an iPhone, SPARC also reads what's playing through Apple Media Service and, between gestures, uses the strip as a progress bar for the current track. Other devices don't offer that service, so they only get the gesture animations.

## Hardware

- ESP32-WROOM-32 module, soldered straight onto the PCB (no devboard)
- VL53L0X time-of-flight distance sensor on I2C
- 8-LED WS2812B (NeoPixel) stick and a status LED
- Single-cell LiPo, charged through a TP4056, regulated to 3.3 V by an AP2112K
- Power slide switch, plus Reset and BOOT buttons
- 4-pin header for a CP2102 USB-serial adapter, used for flashing

The KiCad project for PCB v2 is in `pcb/`, along with a STEP model and `sparc_pcb_v2_gerbers.zip`, the Gerbers we sent to the fab. The case is two printable parts, `enclosure/SPARC_Base.stl` and `enclosure/SPARC_Lid.stl`.

PCB v1 is kept in `pcb/v1/`. Its Reset and BOOT buttons are wired across the power rail, so pressing one shorts 3.3 V to ground and the board can only be flashed with a jumper wire. `pcb/v1/PCB_V1_NOTES.md` covers that workaround and the other v1 bugs, all of which v2 fixes.

The battery level SPARC reports to your phone is hardcoded at 100%. The board doesn't measure it yet.

## Flashing

The firmware is `sketch_esp_hid_v2/sketch_esp_hid_v2.ino`. It needs the ESP32 Arduino core and three libraries: NimBLE-Arduino, Adafruit_VL53L0X and FastLED.

The sensor, LED strip and status LED sit on different pins on each board, so set `BOARD` at the top of the sketch before you build: `2` for PCB v2, `1` for PCB v1, `0` for the old perfboard. The wrong value builds fine, but the sensor and LEDs stay dead.

With the power switch on, connect a CP2102 adapter to the header (the adapter doesn't power the board), hold BOOT, tap Reset, then upload:

```bash
arduino-cli compile --fqbn esp32:esp32:esp32 --port /dev/cu.usbserial-0001 --upload sketch_esp_hid_v2
```

Then pair "SPARC" from your device's Bluetooth settings.

A BLE device stops advertising while it's connected. To pair SPARC with something else, disconnect or forget it on the current device first, or it won't show up.

## Repo layout

- `sketch_esp_hid_v2/` is the current firmware.
- `sketch_esp_hid_v1/` is the earlier HID firmware. It split the zone in two, with tracks close to the sensor and volume farther out.
- `pcb/` holds the PCB v2 design files, and `pcb/v1/` the first revision.
- `enclosure/` holds the case STLs.
- `Main.py`, `ui.py`, `core.py` and `sketch_esp/` are the old desktop app and its firmware. They're no longer used (see below).
- `Wired_version/`, `Uno_version/` and `sketch_nano/` are earlier hardware builds, kept for reference.

## History

SPARC started as an Arduino Uno with an ultrasonic sensor, wired over USB to a Python script that called the Spotify Web API. Our first design mapped hand distance straight to volume. It kept misreading where the hand was, and after about a day of fighting it we switched to discrete passes and holds. Next we went wireless with an HC-05 Bluetooth module, which took a lot of troubleshooting to keep connected on macOS, and then moved to an ESP32 so we no longer needed a separate radio. We also turned the script into a macOS app with a status screen and gesture animations, so nobody had to run it from a terminal.

The Spotify API ended up as a hard limit on the whole project. In February 2026 Spotify cut Development Mode apps to five users and started requiring the developer to have Premium, and extended quota has only been open to organizations since May 2025. We had no way to apply. So we dropped the API and the desktop app, and rebuilt SPARC as a BLE HID device. That got rid of the user cap, the Premium requirement and the app install, and it made iPhone support possible, because iOS never supported the Bluetooth Serial connection the old app depended on.

After that we moved from a breadboard to perfboard, and then to our own PCB and printed case.

A few other problems came up along the way. Early commits leaked Spotify tokens, so we moved credentials into a gitignored `.env` and purged them from history. On a cold power-up the VL53L0X is still booting when the ESP32 first talks to it, and that failure used to hang `setup()` before Bluetooth started, so the board looked dead. The firmware now retries the sensor five times and starts Bluetooth whether or not it answers.

## What's next

- Measure the battery and report the real level instead of 100%
- A pairing gesture, so switching devices doesn't mean digging through Bluetooth settings

## License

MIT
