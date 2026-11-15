# ESP32 IR AC Controller

![Platform](https://img.shields.io/badge/platform-ESP32-blue)
![Framework](https://img.shields.io/badge/framework-ESP--IDF%20%2B%20Arduino-orange)
![Build](https://img.shields.io/badge/build-PlatformIO-green)
![Protocol](https://img.shields.io/badge/control-MQTT-purple)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

An ESP32-based **infrared air-conditioner controller** driven over **MQTT**. It can act as a **Wi-Fi repeater**, **learn** IR codes from any remote, and is configured by a **desktop GUI** (or remotely over MQTT) without ever recompiling the firmware.

> **This repository is documentation only.** It describes how the system is designed, how it behaves and how it is configured. It intentionally contains no source code and no compiled binaries.

## Highlights

| Capability | Summary |
|---|---|
| IR control of an AC | Power, temperature, mode, fan, swing and turbo sent as IR frames |
| MQTT control | Every function is a plain MQTT topic, so it plugs into any home-automation or custom backend |
| Two ways to talk IR | About 70 built-in AC protocols **or** raw codes learned from your own remote |
| Wi-Fi repeater | Optional access point with NAT through the unit's uplink |
| Dual MQTT brokers | Publish to and listen on two brokers at once |
| No-reflash configuration | Settings live in flash and are written by a desktop tool or over MQTT |
| State persistence | Remembers the AC's last state and can re-send it after a power loss |
| Self-healing | Watchdog, heap monitor, bounded timeouts and auto-reboot on prolonged disconnection |

Each unit has a fixed 8-digit **Device ID** (e.g. `00000009`) compiled into the firmware. It appears in every MQTT topic, the MQTT client ID, the default repeater SSID, and determines the repeater subnet. It is deliberately not editable from the configuration tool.

## Contents

1. [System overview](#system-overview)
2. [Hardware](#hardware)
3. [Operating modes](#operating-modes)
4. [Configuration storage (NVS)](#configuration-storage-nvs)
5. [MQTT interface](#mqtt-interface)
6. [Remote configuration over MQTT](#remote-configuration-over-mqtt)
7. [Dual-broker mode](#dual-broker-mode)
8. [Wi-Fi repeater mode](#wi-fi-repeater-mode)
9. [IR learning mode](#ir-learning-mode)
10. [Reliability and self-healing](#reliability-and-self-healing)
11. [Boot behavior and state restore](#boot-behavior-and-state-restore)
12. [Config Writer desktop tool](#config-writer-desktop-tool)
13. [Firmware build system](#firmware-build-system)
14. [Security notes](#security-notes)
15. [Known limitations](#known-limitations)
16. [License](#license)

## System overview

```
                       ┌──────────────────────────────┐
  Home automation /    │         MQTT broker(s)       │
  app / script  ─────► │  (one, or two in dual mode)  │
                       └──────────────┬───────────────┘
                                      │  ac/<id>/...
                       ┌──────────────▼───────────────┐
   Config Writer       │            ESP32             │      IR LED  ───►  Air conditioner
   (desktop, USB) ───► │  Wi-Fi STA (+ optional AP)   │
   writes settings     │  MQTT client(s)              │      IR receiver ◄─ Original remote
   to flash            │  IR sender / learner         │      (learning only)
                       └──────────────┬───────────────┘
                                      │ optional AP + NAT
                                      ▼
                              Downstream Wi-Fi clients
```

Two deliverables:

1. **Firmware** runs on the ESP32: connects to Wi-Fi, subscribes to MQTT topics and turns commands into IR transmissions.
2. **Config Writer** is a Windows desktop program (Python/Tkinter, packaged as a single portable `.exe`) that writes settings into the ESP32's flash over USB and drives IR learning.

## Hardware

| Item | Detail |
|---|---|
| Microcontroller | Classic ESP32 (`esp32dev` board profile) |
| IR transmit pin | **GPIO 13** (drives the IR LED stage) |
| IR receive pin | **GPIO 15** (receiver module, used only while learning) |
| Carrier frequency | 38 kHz for learned/raw playback |
| Serial | 115200 baud over USB |
| Flash size | Configured for **2 MB**. Many dev boards have 4 MB; the build-system flash size and board setting must agree or the partition layout will be wrong |

IR transmission is **software generated** (timed pin toggling), which shapes the reliability design below.

## Operating modes

The `ac_proto` setting selects how the unit speaks to the air conditioner.

### Protocol mode (library-driven)

The unit holds a complete logical AC state (power, temperature, mode, fan, swing, turbo) and encodes the whole state into the correct frame for the chosen brand, like a real remote.

- Default protocol: `COOLIX`. Unknown names log a warning and fall back to `COOLIX`.
- Supported families include Daikin, Mitsubishi, Panasonic, Samsung, LG, Gree, Midea, Haier, Hitachi, Fujitsu, Toshiba, Carrier, Sharp, Whirlpool, Electra, Kelvinator, TCL, Vestel, Voltas, York and more.
- Valid temperature range: **20–30**. Turbo is available in this mode only.

### LEARNED mode

For ACs not covered by a built-in protocol, record the real remote's buttons and replay them as raw timing data. `ac_proto` is set to `LEARNED` automatically when learned codes are saved.

- **14 button slots**: power on/off, temp up/down, mode cool/heat/dry/fan/auto, fan auto/min/med/max, swing toggle.
- Temperature is reached by **stepping**: one up/down press every 400 ms until the tracked setpoint matches, running in the background so MQTT handling never blocks.
- Swing is a single toggle; the unit tracks its state so `on` and `off` behave as absolute commands.
- Turbo is not available. Missing slots are skipped safely.

## Configuration storage (NVS)

All settings are stored in the ESP32's NVS partition (offset `0x9000`, size `0x5000`).

### Namespace `cfg`

| Key | Type | Meaning |
|---|---|---|
| `ssid` | string (≤32) | Wi-Fi network to join (required) |
| `pass` | string (≤64) | Wi-Fi password |
| `mqtt_host1` | string (≤64) | Broker 1 host (required) |
| `mqtt_port1` | u16 | Broker 1 port (default 1883) |
| `mqtt_user1` / `mqtt_pass1` | string | Broker 1 credentials (empty = anonymous) |
| `dual_mode` | u8 | `1` enables the second broker |
| `mqtt_host2`, `mqtt_port2`, `mqtt_user2`, `mqtt_pass2` | as above | Broker 2 settings |
| `repeater_en` | u8 | `1` enables the repeater AP (default off) |
| `ap_ssid` | string (≤32) | Repeater network name (default `AC-<device id>`) |
| `ap_pass` | string (≤64) | Repeater password (min 8 chars for WPA2) |
| `ac_proto` | string | Protocol name or `LEARNED` |

### Other namespaces (firmware-managed)

| Namespace | Holds |
|---|---|
| `acstate` | Last AC state in protocol mode |
| `irlearn` | Learned raw IR codes, tracked setpoint, power and swing state |
| `bootcfg` | Boot-resync on/off switch |

### Boot sanity checks

The firmware warns and degrades safely when it finds: no SSID; dual mode without a second broker host; repeater enabled with a password under 8 characters (the AP refuses to start and never falls back to open); an MQTT password with no username; or a restored temperature outside 20–30 (clamped to 24).

## MQTT interface

All topics are prefixed `ac/<id>/`. Subscriptions use QoS 1; command payloads are case-insensitive.

### Commands (you publish)

| Topic | Payloads |
|---|---|
| `ac/<id>/set/power` | `on`, `off` |
| `ac/<id>/set/temp` | integer `20`–`30` |
| `ac/<id>/set/mode` | `cool`, `heat`, `dry`, `fan`, `auto` |
| `ac/<id>/set/fan` | `auto`, `min`, `med`, `max` |
| `ac/<id>/set/swing` | `on`, `off`, `toggle` |
| `ac/<id>/set/turbo` | `on`, `off` (protocol mode only) |
| `ac/<id>/set/bootresync` | `on`/`off` (also `1`/`0`, `true`/`false`) |

Invalid payloads are ignored, as are all commands while an IR-learning capture is armed.

### Feedback (the unit publishes)

| Topic | Retained | Content |
|---|---|---|
| `ac/<id>/state` | Yes | JSON snapshot of the AC state |
| `ac/<id>/ack` | No | JSON acknowledgement of the last accepted command |
| `ac/<id>/status` | Yes | `online` (refreshed every 60 s); broker publishes `offline` as last-will |
| `ac/<id>/bootresync` | Yes | Current boot-resync setting |
| `ac/<id>/cfgresult` | No | Result of remote configuration commands |
| `ac/<id>/cfgstate/<key>` | No | Current configuration values, on request |

**State fields:** protocol mode: `device`, `power`, `degrees`, `mode`, `fan`, `swing`, `turbo`. LEARNED mode: `device`, `power`, `degrees`, `swing`. The snapshot is re-published on every broker (re)connection.

**Ack fields:** `device`, `topic`, `value`, `power`, `uptime_ms`.

## Remote configuration over MQTT

Settings can be changed without a USB cable. Changes are **staged in RAM, then committed together**, so configurations are never half-applied.

| Topic (publish) | Payload | Effect |
|---|---|---|
| `ac/<id>/cfg/set/<key>` | the value | Validates and stages one setting |
| `ac/<id>/cfg/commit` | anything | Writes all staged settings to flash, then **reboots** |
| `ac/<id>/cfg/cancel` | anything | Discards staged settings |
| `ac/<id>/cfg/get` | anything | Publishes current settings (passwords shown only as set/unset) |

- Every command gets an `ok:...` or `err:...` reply on `cfgresult`.
- To set a text value to empty, send the literal `<empty>` (an empty payload clears retained messages).
- Config commands are cleared from the broker's retained storage on receipt, so a retained `commit` can never cause a reboot loop.
- Over-length values are rejected; case is preserved.

## Dual-broker mode

With `dual_mode` on, the unit keeps two independent MQTT connections.

- Commands from **either** broker are accepted.
- Acks, state and config replies go to the originating broker **and** the other one.
- Each connection has its own exponential back-off retry timer.
- If broker 2 has no host, the unit falls back to single-broker operation.
- Boot re-sync fires as soon as **either** broker connects.

## Wi-Fi repeater mode

When `repeater_en` is `1`, the unit runs **AP + STA** simultaneously: the station joins your Wi-Fi, and the access point (up to 8 clients, WPA2, channel follows the uplink) routes traffic out through it using **NAT**, passing upstream DNS to clients via DHCP.

- **Off by default**, so no stray default-named network appears.
- **Unique subnet per device:** `192.168.X.0/24` with `X = (ID mod 250) + 2`, so repeater chains never collide.
- NAT and DNS are re-established after every uplink reconnection.
- Reconnection never powers the radio off, so downstream clients stay connected.
- Requires specific network-stack build options (see [Firmware build system](#firmware-build-system)).

## IR learning mode

Learning is driven from the Config Writer over USB serial using a small line-based protocol.

### Workflow

1. **Probe (optional):** press any button on the original remote. The unit reports `MATCH:<name>` (use protocol mode) or `UNKNOWN`.
2. **Capture each button:** choose a slot; the receiver arms for **15 seconds** and the capture is **staged in RAM**.
3. **Commit:** all staged slots are written to flash, `ac_proto` becomes `LEARNED`, and the unit reboots.

A commit is a **full replace**: slots not re-captured in the session are cleared. `ABORT` discards staged captures; `CANCEL` stops an armed capture.

### Serial protocol

Commands are lines sent to the unit; replies begin with `RSP:LEARN:`.

| Command | Purpose |
|---|---|
| `CMD:LEARN:PROBE` | Identify a remote's protocol |
| `CMD:LEARN:SLOT:<key>` | Arm capture for one slot |
| `CMD:LEARN:CANCEL` | Stop the armed capture |
| `CMD:LEARN:COMMIT` | Save staged codes and reboot |
| `CMD:LEARN:ABORT` | Discard staged codes |
| `CMD:LEARN:STATUS` | Report staged and saved counts |
| `CMD:LEARN:LIST` | List slot keys and labels |

Responses: `ARMED`, `MATCH`, `UNKNOWN`, `CAPTURED`, `FAILED`, `TIMEOUT`, `STATUS`, `STAGED`, `COMMITTING`, `SAVED`, `CANCELLED`, `ABORTED`, `ERROR`.

Slot keys: `pw1`, `pw0`, `tup`, `tdn`, `mcl`, `mht`, `mdr`, `mfn`, `mau`, `fau`, `fmn`, `fmd`, `fmx`, `swt`.

### IR profiles

The Config Writer can save captured codes as a named **profile** and write it onto other units that have no IR receiver, so one unit with a receiver can "teach" a whole fleet.

## Reliability and self-healing

| Mechanism | What it does |
|---|---|
| Task watchdog (60 s) | Watches the main loop only (idle tasks excluded to avoid false resets under Wi-Fi/NAT load); a hang triggers a clean reboot |
| Heap monitor | Logs free/fragmented heap every 30 s; reboots below 20 KB free |
| Disconnect timeout | Reboots if Wi-Fi or a broker is down for more than 5 minutes |
| Back-off retries | Reconnect intervals double from 1 s up to 30 s |
| Bounded TCP connect | Explicit 3 s timeout, so an unreachable broker can't outlast the watchdog |
| Non-blocking temp ramp | One step per loop pass instead of blocking in the MQTT handler |
| IR blast window | Loop task priority is raised during transmit so network processing can't corrupt pulse timing, then restored |
| Learning guard | Commands ignored while a capture is armed |
| Wi-Fi tuning | Power-save off, maximum transmit power |

## Boot behavior and state restore

1. Load configuration, learned IR codes and the boot-resync setting from flash.
2. Build topic names, start the watchdog, initialise IR pins.
3. Restore the last AC state (protocol mode from `acstate`; learned mode from `irlearn`).
4. Start Wi-Fi (and the repeater AP if enabled), then connect to broker(s).
5. **Boot re-sync:** once a broker connects, the restored state is re-sent over IR. This covers an AC that lost power and reset to defaults while the controller remembered the old state.

Boot re-sync is on by default, stored in flash, and switchable **only via MQTT** (`set/bootresync`), not from the desktop tool.

## Config Writer desktop tool

A Tkinter GUI that writes the `cfg` namespace into the ESP32 over USB, with no rebuild or reflash of the firmware.

**Features**

- Serial port detection with ESP32 adapter hints and auto-refresh
- Form-based configuration of Wi-Fi, one or two MQTT brokers, repeater and AC protocol (dropdown incl. `LEARNED`)
- Live validation of hostnames and ports; required-field check before writing
- Password visibility toggles
- **Write config:** builds an NVS image, flashes it at the NVS offset, resets the board
- **Read config from device:** reads and decodes the NVS partition back into the form
- Live serial monitor with colour-coded log, zoom, copy and clear; device ID auto-detected from the boot message
- IR learning window (probe, per-button capture, commit, abort)
- IR profiles: save a device's captured codes and apply them to other devices
- Responsive layout with a draggable divider

**How a write works**

1. The serial monitor releases the port.
2. The existing NVS partition is **read first** and all other namespaces (learned codes, saved AC state) are preserved, so changing an SSID never wipes learned data.
3. Only `cfg` is replaced with the form values. If a profile is selected, its codes replace `irlearn` and `ac_proto` is forced to `LEARNED`.
4. An NVS image is generated and flashed at 460800 baud, then the board resets.
5. Blank optional repeater fields are not written, so firmware defaults apply.

Only one flash operation runs at a time.

**Building the Windows executable**

Packaged as one portable `.exe` with PyInstaller via a small build script. You need Python 3 on the PATH; the script checks the required files, creates a virtual environment, installs dependencies (serial library, ESP flashing library, PyInstaller, NVS image generator), and builds a single windowed executable. The result runs on any Windows PC with no Python, pip or Arduino tooling. SmartScreen will warn because the executable is unsigned ("More info", then "Run anyway"). IR profiles are stored in an `ir_profiles` folder next to the executable.

## Firmware build system

| Aspect | Detail |
|---|---|
| Toolchain | PlatformIO |
| Platform / board | Espressif 32, `esp32dev` |
| Framework | **ESP-IDF + Arduino (hybrid)**, because repeater NAT isn't available in the prebuilt Arduino core |
| Libraries | An MQTT client and an IR send/receive/AC library |
| Monitor | 115200 baud with exception decoding |
| Logging | Library debug off; default log level none |

**Notable build options**

| Option | Why |
|---|---|
| IP forwarding, NAPT, L2→L3 packet copy | Needed for the repeater to route (NAT rewrites packets, so zero-copy is incompatible) |
| Up to 8 DHCP clients | Headroom for the DHCP server and NAT tables |
| 1000 Hz FreeRTOS tick | **Required** by the Arduino component |
| Auto-start Arduino | **Required**, otherwise the build fails with `undefined reference to app_main` |
| Task watchdog 60 s | Matches the application's watchdog timeout |
| Flash size 2 MB | Must match in the build-system board setting **and** ESP-IDF setting; change all to 4 MB for 4 MB boards |

The component build file limits source discovery to real source extensions so non-source files in the source folder aren't compiled.

## Security notes

- **Default repeater password:** if none is stored, a well-known built-in default is used. Always set your own before enabling the repeater.
- **MQTT is unencrypted** (plain TCP, no TLS). Keep brokers on a trusted network or behind a VPN.
- **Credentials are stored in plain text** in flash; flash encryption is needed to protect against physical access.
- **Remote configuration is powerful:** anyone who can publish to `ac/<id>/cfg/#` can change Wi-Fi settings and reboot the unit. Restrict it with broker ACLs.
- **No OTA updates;** updates require USB.
- **Device IDs are sequential and guessable.** Don't treat topic names as secrets.

## Known limitations

- The device ID is fixed at compile time, so each unit needs its own firmware build.
- Learned mode can't control turbo, and temperature is stepped relatively, so the tracked setpoint can drift if the physical remote is also used.
- IR is one-way: the controller **cannot read** the AC's real state, only report what it last sent.
- IR timing is software generated, so extreme network load can still affect it despite the priority boost.
- The repeater is NAT-based (a routed second network), not a transparent L2 bridge.
- Only 14 button slots are learnable, and only one profile can be applied per write.

## License

Released under the [MIT License](LICENSE).
